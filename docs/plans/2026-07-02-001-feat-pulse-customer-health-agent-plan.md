---
title: "feat: Pulse — Open-Source Customer Health Check-In Agent"
date: 2026-07-02
origin: /Users/shasha/Desktop/pulse-agent-spec.md
type: feat
depth: standard
---

# Pulse — Open-Source Customer Health Check-In Agent

> **TL;DR:** Build and open-source Pulse, a customer health monitoring agent that connects to Stripe, calculates health scores, sends automated check-in emails, and flags at-risk accounts — all running on Cloudflare Workers + D1 for ~$3-5/mo infrastructure cost.

---

## Summary

Pulse is an open-source (MIT) AI agent that monitors SaaS customer health across Stripe and product usage data. It runs on Cloudflare Workers + D1 + Cron Triggers + Email Service, with a minimal HTML dashboard via Cloudflare Pages. The entire infrastructure costs ~$3-5/mo.

The repo will be at `github.com/<user>/pulse`.

---

## Problem Frame

Small SaaS teams ($2k-$200k MRR, 1-20 employees) lose customers to churn because:

1. They don't know which customers are at risk until they cancel
2. They don't have time to manually check in on quiet customers
3. Enterprise CS platforms (Gainsight $10k+/mo, Vitally $3k+/mo, Custify $899/mo) are 10-100x too expensive
4. General messaging tools (Customer.io, Intercom) require manual setup and don't have built-in health scoring

Pulse solves this by being a focused, affordable agent that just does one thing: watch customer health and reach out before they churn.

---

## Requirements

### Functional Requirements

1. **R1**: Connect to Stripe and sync customer/subscription data (via webhooks + initial sync)
2. **R2**: Calculate a health score per customer based on engagement, support health, and payment health
3. **R3**: Classify customers into Green/Yellow/Red tiers based on health score
4. **R4**: Run scheduled health scans via Cron Triggers (weekly default, configurable)
5. **R5**: Send automated check-in emails to Yellow-tier customers
6. **R6**: Flag Red-tier customers for human review with escalation rules
7. **R7**: Display a dashboard showing health tiers, at-risk customers, and activity log
8. **R8**: Allow users to review and approve draft emails before auto-send (v1)
9. **R9**: Track intervention outcomes (login rates, response rates, MRR saved)

### Non-Functional Requirements

10. **R10**: Multi-tenant — support multiple Pulse users with isolated data
11. **R11**: Authentication via Cloudflare Access
12. **R12**: Total infrastructure cost under $10/mo at launch scale (<100 customers per tenant)
13. **R13**: Open-source (MIT license)

---

## Key Technical Decisions

### KTD1: Multiple Workers Architecture

**Decision:** Split into `api` Worker (Stripe webhooks + REST API) and `scanner` Worker (cron health scans).

**Rationale:** The scanner runs on a cron schedule with no HTTP traffic. Keeping it separate means:
- Each Worker has minimal permissions (scanner doesn't need HTTP routes)
- Cron failures don't affect API availability
- Independent deploy and scale

**Trade-off:** Two `wrangler.toml` files and two deployments instead of one.

### KTD2: Cloudflare Access for Auth

**Decision:** Use Cloudflare Access as the auth layer for the dashboard and API.

**Rationale:** Zero auth code to write, maintain, or audit. Cloudflare Access handles SSO, MFA, session management. Users authenticate via their Cloudflare account. The dashboard Worker just checks for the `Cf-Access-Authenticated-User-Email` header.

**Trade-off:** Users need a Cloudflare account and the domain must be proxied through Cloudflare.

### KTD3: Minimal HTML/JS Dashboard via Cloudflare Pages

**Decision:** Vanilla HTML + CSS + JS served as static assets from Cloudflare Pages, calling the `api` Worker for data.

**Rationale:** Fastest to build (no framework setup), zero bundle size, easy to contribute to, no build step for the frontend. The dashboard is simple — 3 tabs of data with forms for approve/send actions.

**Trade-off:** No component reusability or reactive framework. Acceptable for this scope.

### KTD4: D1 for Storage

**Decision:** Cloudflare D1 (SQLite) for all persistent data.

**Rationale:** Free tier covers launch (<5M reads/mo). SQL schema is straightforward (4-5 tables). Integrated with Workers via binding — no connection pooling, no VPC, no ORM needed.

**Trade-off:** Not ideal for write-heavy workloads or complex queries. Fine for Pulse's read-heavy, scan-based workload.

### KTD5: Cloudflare Email Service for Email

**Decision:** Use Cloudflare's Email Service binding for sending check-in emails.

**Rationale:** Included in Workers pricing. No third-party email API costs. Up to 2,000 emails/day free with a verified domain.

**Trade-off:** Not suitable for bulk/marketing volume. Perfect for Pulse's transactional check-in volume.

---

## Scope Boundaries

### In Scope (v1)

- Stripe integration via webhooks (customer.created, customer.updated, customer.subscription.updated, invoice.payment_*)
- Health score engine (engagement + payment + support signals)
- Weekly cron-triggered health scans
- Email check-in sending via Cloudflare Email Service
- Dashboard with health overview, at-risk list, and activity log
- Multi-tenant data isolation (D1 per tenant or tenant-scoped rows)
- Cloudflare Access auth
- MIT open-source license
- Deployment guide and README

### Deferred to Follow-Up Work

- Intercom/Zendesk support ticket integration (v1 uses Stripe-only scoring)
- Product usage analytics API integration (v1 uses Stripe subscription age + payment history as proxy)
- Slack alerts for Red-tier escalations
- Recovery offer flow (auto-discounting)
- Outcome tracking dashboard (MRR saved, ROI)
- Custom health score formulas per tenant
- Self-hosted deployment guide (non-Cloudflare)

### Deferred for Later (Product Expansion)

- Outcome-based pricing model (10% of recovered MRR)
- Paid tiers (Pulse Lite/Pro/Scale)
- Multi-user team accounts
- API for external integrations

---

## Open Questions

- **O1**: Exact Stripe webhook events to subscribe to — resolved during implementation based on Stripe webhook docs
- **O2**: Whether to use a single D1 database with tenant-scoped rows or per-tenant databases — decision deferred to implementation based on scale expectations
- **O3**: Email template wording — iterate during pilot; v1 uses the drafts from the spec

---

## Implementation Units

### U1. Project Scaffolding

- **Goal:** Initialize the Pulse repo with monorepo structure, wrangler configs, package.json, and MIT license
- **Requirements:** R13 (open-source)
- **Dependencies:** None
- **Files:**
  - `README.md` — create
  - `LICENSE` — create (MIT)
  - `package.json` — create (workspaces config)
  - `.gitignore` — create
  - `packages/api/wrangler.toml` — create (Worker config, D1 binding, Email binding, Stripe secret)
  - `packages/api/src/index.ts` — create (skeleton with fetch handler + route dispatch)
  - `packages/api/src/db.ts` — create (D1 client wrapper)
  - `packages/api/package.json` — create
  - `packages/scanner/wrangler.toml` — create (cron trigger config, D1 binding, Email binding)
  - `packages/scanner/src/index.ts` — create (skeleton with scheduled handler)
  - `packages/scanner/package.json` — create
  - `packages/dashboard/public/index.html` — create (minimal skeleton)
  - `packages/dashboard/public/styles.css` — create (empty)
  - `packages/dashboard/public/app.js` — create (empty)
  - `packages/dashboard/wrangler.toml` — create (Pages config)
  - `packages/shared/src/schema.ts` — create (TypeScript types)
  - `packages/shared/src/migrations/001_init.sql` — create (empty, scaffolded)
  - `packages/shared/package.json` — create
  - `.github/workflows/deploy.yml` — create (CI/CD deploy)
- **Approach:**
  1. Create the directory structure with `packages/api`, `packages/scanner`, `packages/dashboard`, `packages/shared`
  2. Use npm workspaces in root `package.json`
  3. Each Worker package has its own `wrangler.toml` referencing shared D1 database and email bindings
  4. Scaffold worker entry points with basic request/event handling
- **Patterns to follow:** Cloudflare Workers patterns — `export default { async fetch(request, env, ctx) { ... } }` for HTTP workers, `export default { async scheduled(controller, env, ctx) { ... } }` for cron workers
- **Test scenarios:**
  - Repo structure is valid: workspaces resolve, wrangler configs parse
  - Each Worker entry point can be imported without errors
  - TypeScript compiles with `tsc --noEmit`

### U2. D1 Schema & Shared Types

- **Goal:** Define the database schema and shared TypeScript types for customers, health scores, and check-ins
- **Requirements:** R1, R2, R3, R10 (multi-tenant)
- **Dependencies:** U1
- **Files:**
  - `packages/shared/src/migrations/001_init.sql` — modify (full schema)
  - `packages/shared/src/types.ts` — modify (TypeScript interfaces for DB rows + API responses)
  - `packages/api/src/db.ts` — modify (typed query helpers)
  - `packages/scanner/src/db.ts` — create (scanner-specific query helpers if different from API)
- **Approach:**
  - **Schema design:**
    ```sql
    -- Each Pulse "user" gets a tenant_id (UUID)
    -- Data is scoped by tenant_id for multi-tenancy

    CREATE TABLE tenants (
      id TEXT PRIMARY KEY,
      stripe_account_id TEXT UNIQUE,
      email_from TEXT NOT NULL,     -- verified sender email
      config JSON,                  -- health score weights, escalation rules
      created_at TEXT NOT NULL DEFAULT (datetime('now')),
      updated_at TEXT NOT NULL DEFAULT (datetime('now'))
    );

    CREATE TABLE customers (
      id TEXT PRIMARY KEY,           -- Stripe customer ID
      tenant_id TEXT NOT NULL REFERENCES tenants(id),
      email TEXT NOT NULL,
      name TEXT,
      plan_type TEXT,
      mrr INTEGER DEFAULT 0,        -- in cents
      status TEXT DEFAULT 'active',  -- active, past_due, canceled
      last_login_at TEXT,
      support_tickets_30d INTEGER DEFAULT 0,
      created_at TEXT NOT NULL,
      updated_at TEXT NOT NULL
    );
    CREATE INDEX idx_customers_tenant ON customers(tenant_id);

    CREATE TABLE health_scores (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      customer_id TEXT NOT NULL REFERENCES customers(id),
      tenant_id TEXT NOT NULL REFERENCES tenants(id),
      score INTEGER NOT NULL,        -- 0-100
      tier TEXT NOT NULL,            -- green, yellow, red
      engagement_points INTEGER,
      support_points INTEGER,
      payment_points INTEGER,
      scanned_at TEXT NOT NULL
    );
    CREATE INDEX idx_health_scores_customer ON health_scores(customer_id, scanned_at);

    CREATE TABLE check_ins (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      customer_id TEXT NOT NULL REFERENCES customers(id),
      tenant_id TEXT NOT NULL REFERENCES tenants(id),
      type TEXT NOT NULL,             -- check_in, recovery, escalation
      status TEXT NOT NULL DEFAULT 'draft',  -- draft, sent, approved, rejected
      subject TEXT,
      body TEXT,
      sent_at TEXT,
      replied_at TEXT,
      login_after_send INTEGER,      -- did they log in within 48h? 0/1
      created_at TEXT NOT NULL DEFAULT (datetime('now'))
    );
    CREATE INDEX idx_check_ins_tenant ON check_ins(tenant_id, status);
    ```
  - Shared types mirror the schema + API response wrappers
  - DB helpers in `api/src/db.ts` provide typed wrappers over `env.DB.prepare()`
- **Test scenarios:**
  - Migrations apply successfully against D1 dev environment
  - Can insert, read, update customers with correct types
  - Foreign key relationships work (cascade behavior)
  - Index queries perform as expected with small datasets

### U3. API Worker — Stripe Webhook Handler

- **Goal:** Receive Stripe webhook events, sync customer/subscription data into D1, serve the REST API for the dashboard
- **Requirements:** R1 (Stripe integration), R8 (draft/approve flow)
- **Dependencies:** U2
- **Files:**
  - `packages/api/src/index.ts` — modify (add stripe webhook route, add REST API routes)
  - `packages/api/src/stripe.ts` — create (Stripe webhook signature verification + event handlers)
  - `packages/api/src/routes/customers.ts` — create (GET /api/customers, GET /api/customers/:id)
  - `packages/api/src/routes/checkins.ts` — create (GET /api/checkins, POST /api/checkins/:id/approve, POST /api/checkins/:id/reject)
  - `packages/api/src/routes/health.ts` — create (GET /api/overview — aggregated health stats)
  - `packages/api/src/auth.ts` — create (Cloudflare Access header validation middleware)
- **Approach:**
  - Stripe webhook endpoint at `POST /webhooks/stripe`
  - Verify signature using Stripe's raw body requirement: read `request.body` as text, verify with `stripe.webhooks.constructEvent()` (using webcrypto polyfill for Cloudflare Workers)
  - Subscribe to events: `customer.created`, `customer.updated`, `customer.subscription.updated`, `invoice.payment_succeeded`, `invoice.payment_failed`
  - REST API endpoints gated by Cloudflare Access header check (validate `Cf-Access-Authenticated-User-Email` exists and matches a tenant)
  - REST API:
    - `GET /api/overview` — tier counts, MRR by tier, recent activity summary
    - `GET /api/customers` — list customers with latest health scores, filterable by tier
    - `GET /api/customers/:id` — single customer detail + score history
    - `GET /api/checkins` — recent check-ins, filterable by status
    - `POST /api/checkins/:id/approve` — approve a draft check-in for sending
    - `POST /api/checkins/:id/reject` — reject a draft with optional feedback
- **Test scenarios:**
  - Stripe webhook: valid signature processes event; invalid signature rejects with 401
  - Stripe webhook: `customer.subscription.updated` with status "canceled" correctly marks customer status
  - REST API: unauthenticated request returns 401
  - REST API: valid Cloudflare Access header returns data for correct tenant only
  - REST API: customer endpoint returns paginated, filterable results
  - REST API: approve check-in sets status to "approved", returns 200
  - Edge case: duplicate Stripe events are idempotent

### U4. Scanner Worker — Health Score Engine

- **Goal:** Run on a cron schedule, calculate health scores for all active customers, generate draft check-ins
- **Requirements:** R2 (health score), R3 (tier classification), R4 (cron scans), R5 (check-in emails), R6 (escalation routing)
- **Dependencies:** U2 (schema), U3 (for customer data)
- **Files:**
  - `packages/scanner/src/index.ts` — modify (scheduled handler with scan orchestration)
  - `packages/scanner/src/scanner.ts` — create (health score calculation logic)
  - `packages/scanner/src/checkins.ts` — create (draft check-in generation using email templates)
  - `packages/scanner/src/templates.ts` — create (email templates: check-in, recovery, feedback)
- **Approach:**
  - Cron trigger: `0 9 * * 1` (every Monday at 9 AM) — configurable via env var
  - Scan process:
    1. Fetch all active customers for the tenant
    2. For each customer, calculate health score:
       - Engagement (50 pts): based on `last_login_at` recency (if populated) or subscription age proxy
       - Support health (25 pts): based on `support_tickets_30d` count
       - Payment health (25 pts): based on Stripe subscription status + invoice payment events
    3. Insert health score snapshot into `health_scores` table
    4. Determine tier: Green (75-100), Yellow (40-74), Red (0-39)
    5. For Yellow tier: generate draft check-in email, insert into `check_ins` with status "draft"
    6. For Red tier: generate escalation entry, mark for human review
    7. For Red tier with MRR > $500: send immediate notification (future: Slack webhook — v1 just flags in dashboard)
  - Email templates are parameterized: customer name, days since login, offer text
  - Templates stored as template strings in `templates.ts`
- **Test scenarios:**
  - Customer with last_login=2d, 0 tickets, current payment → score 100, tier green
  - Customer with last_login=23d, 0 tickets, current payment → score 70, tier yellow, draft created
  - Customer with last_login=45d, 1 unresolved ticket, past_due → score 5, tier red, escalation flagged
  - Customer with last_login=1d, 0 tickets, current payment → no action (green)
  - Edge case: customer with null last_login uses subscription age proxy
  - Edge case: tenant with 0 customers exits gracefully
  - Edge case: same customer scanned twice in a day inserts new snapshot without error

### U5. Email Integration

- **Goal:** Send check-in emails via Cloudflare Email Service, track delivery and replies
- **Requirements:** R5 (email sending)
- **Dependencies:** U2 (schema), U4 (draft generation), U3 (approval check-in from API)
- **Files:**
  - `packages/api/src/email.ts` — create (email sending via Email Service binding)
  - `packages/api/src/routes/checkins.ts` — modify (on approve, trigger send instead of just updating status)
  - `packages/scanner/src/email.ts` — create (or shared import from api)
  - `packages/scanner/wrangler.toml` — modify (add email binding config)
  - `packages/api/wrangler.toml` — modify (add email binding config)
- **Approach:**
  - Cloudflare Email Service binding: `EMAIL` binding in `wrangler.toml`
  - Send function:
    ```typescript
    async function sendEmail(env: Env, to: string, subject: string, body: string) {
      await env.EMAIL.send({
        from: { email: env.EMAIL_FROM, name: env.EMAIL_FROM_NAME },
        to: [{ email: to }],
        subject,
        htmlBody: body,
      });
    }
    ```
  - Integration with API: when a check-in is approved via `POST /api/checkins/:id/approve`, the API Worker sends the email immediately and updates `sent_at`
  - Integration with scanner: in v1, scanner creates drafts only; a separate scheduled send Worker could be added later for auto-send mode
  - Reply tracking: v1 uses a simple reply-to address; future versions can use inbound email handler
- **Test scenarios:**
  - Approved check-in sends email successfully
  - Invalid email address returns appropriate error
  - Email body renders correctly (no template interpolation errors)
  - HTML email renders in common email clients (manual review)

### U6. Dashboard — Minimal HTML/CSS/JS

- **Goal:** Serve a functional dashboard from Cloudflare Pages showing health overview, at-risk customers, and activity log
- **Requirements:** R7 (dashboard), R9 (track outcomes)
- **Dependencies:** U3 (API Worker serving data)
- **Files:**
  - `packages/dashboard/public/index.html` — modify (full dashboard HTML)
  - `packages/dashboard/public/styles.css` — modify (dashboard CSS)
  - `packages/dashboard/public/app.js` — modify (dashboard JS — fetch API, render tabs, handle actions)
  - `packages/dashboard/wrangler.toml` — modify (proxy API routes to the api Worker)
- **Approach:**
  - Three-tab layout using tab buttons and div show/hide (no router)
    - Tab 1: **Overview** — tier breakdown table (Green/Yellow/Red with counts + MRR), weekly activity summary, "Pulse sent X check-ins, Y re-engaged" stats
    - Tab 2: **At-Risk** — list of Yellow/Red customers with health score, last login, MRR. For each: "View Draft → Edit → Approve / Reject" buttons. Draft email preview in a modal or expandable section.
    - Tab 3: **Activity Log** — chronological list of check-ins with status, response, outcome
  - CSS: minimal, clean, no framework. Use system font stack, simple color palette (green/yellow/red for tiers).
  - JS: vanilla `fetch()` calls to the API Worker, DOM manipulation to render tables. No build step.
  - Cloudflare Pages proxies `/api/*` requests to the API Worker via `wrangler.toml` routes
- **Test scenarios:**
  - Dashboard loads without JS errors
  - Each tab renders correct data from API
  - Approve button sends POST to API and updates UI
  - Reject button works similarly
  - Empty state renders correctly (no customers, no check-ins)
  - Error state handles API failure gracefully

### U7. Onboarding Flow

- **Goal:** Allow a new Pulse user to set up their tenant, connect Stripe, and configure email sender
- **Requirements:** R1 (Stripe integration setup), R11 (Cloudflare Access), R10 (tenant creation)
- **Dependencies:** U3 (API Worker), U5 (email setup), U6 (dashboard)
- **Files:**
  - `packages/api/src/routes/onboarding.ts` — create (POST /api/onboarding — tenant creation, Stripe Connect link)
  - `packages/api/src/index.ts` — modify (add onboarding route, no auth required for initial step)
  - `packages/dashboard/public/onboarding.html` — create (onboarding wizard page)
  - `packages/api/src/stripe.ts` — modify (add Stripe Connect account link generation)
  - `README.md` — modify (add setup instructions section)
- **Approach:**
  - Cloudflare Access handles auth — user hits the dashboard URL, Cloudflare Access prompts login, on success the request has `Cf-Access-Authenticated-User-Email` header
  - On first visit with no tenant for the user's email, redirect to onboarding page
  - Onboarding steps:
    1. Enter email sender details (verified in Cloudflare Email)
    2. Stripe Connect OAuth flow to link Stripe account
    3. Configure basic preferences (scan frequency, escalation thresholds)
  - On completion, create `tenants` row and redirect to main dashboard
  - Stripe events will start flowing once the webhook endpoint is configured
- **Test scenarios:**
  - New user flow: first visit → onboarding → Stripe Connect → dashboard
  - Returning user: visits dashboard, already has tenant, sees data
  - Stripe Connect callback creates tenant correctly
  - Onboarding with invalid email config shows error

### U8. Deployment & CI

- **Goal:** Enable one-command deployment of all Workers and Pages to Cloudflare
- **Requirements:** R13 (open-source — others need to deploy too)
- **Dependencies:** All other units
- **Files:**
  - `.github/workflows/deploy.yml` — modify (full CI/CD pipeline)
  - `wrangler.toml` (root) — modify (add env configs for staging/production if desired)
  - `packages/api/wrangler.toml` — modify (finalize config with env vars)
  - `packages/scanner/wrangler.toml` — modify (finalize config with cron + env vars)
  - `packages/dashboard/wrangler.toml` — modify (finalize Pages config)
  - `README.md` — modify (complete deployment guide with step-by-step instructions)
- **Approach:**
  - GitHub Actions workflow: on push to `main`, deploy Workers + Pages
  - Wrangler configs reference Cloudflare secrets via `wrangler secret` or GitHub Actions secrets
  - README includes:
    - Prerequisites (Cloudflare account, domain, Stripe account)
    - Step 1: Clone repo
    - Step 2: `npm install`
    - Step 3: Configure `wrangler.toml` files (or use `.env.example`)
    - Step 4: Run `npx wrangler d1 migrations apply pulse-db`
    - Step 5: Deploy with `npx wrangler deploy`
    - Step 6: Configure Stripe webhook endpoint
    - Step 7: Set up Cloudflare Access
  - Environment variables:
    - `STRIPE_SECRET_KEY` — Stripe secret key
    - `STRIPE_WEBHOOK_SECRET` — Stripe webhook signing secret
    - `EMAIL_FROM` — verified sender email
    - `EMAIL_FROM_NAME` — sender name
    - `CLOUDFLARE_ACCESS_AUD` — Access audience tag (for token validation)
- **Test scenarios:**
  - `wrangler deploy` succeeds for each Worker
  - `wrangler d1 migrations apply` runs without errors
  - Stripe webhook endpoint returns 200 on valid ping event
  - Dashboard is accessible via Pages URL
  - README instructions are reproducible (test on a clean environment)

---

## System-Wide Impact

- **New repository:** `pulse` — separate from any existing projects
- **Cloudflare resources required:** Workers (api + scanner), D1 database, Email Service domain, Pages project, Access application
- **Stripe resources required:** Stripe account with webhook endpoint configured
- **Costs:** ~$3-5/mo at launch scale (Workers free tier, D1 ~$1-3/mo, Email free up to 2k/day)
- **Monitoring:** Workers logs available in Cloudflare dashboard; no external monitoring in v1

---

## Risks & Dependencies

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Stripe webhook signature verification tricky on Workers (raw body access) | Medium | High | Document the rawBody access pattern from Cloudflare docs; use Stripe's webcrypto polyfill |
| Cloudflare Email Service delivery reliability (spam, throttling) | Low | Medium | Include DNS setup instructions (SPF, DKIM); test delivery early |
| D1 query performance with growing data | Low | Medium | Schema indexed by tenant_id; health_scores partitioned by date in v2 if needed |
| Multi-tenant data leaks | Low | Critical | Every query scoped by tenant_id; Cloudflare Access prevents cross-tenant auth |
| Low adoption for open-source project | Medium | Low (open source) | Content-driven distribution (churn teardowns, case studies); clear README |

---

## Sources & Research

- Cloudflare Workers Cron Triggers: `docs/workers/configuration/cron-triggers/` — supports standard cron expressions
- Cloudflare Email Service Workers API: `docs/email-service/api/send-emails/workers-api/` — `EMAIL.send()` binding
- Cloudflare D1 documentation: SQLite-compatible, `env.DB.prepare()` interface
- Stripe webhook best practices: verify signature, idempotent event processing, retry handling
- Competitive landscape: Custify ($899/mo), Vitally ($3k+/mo), Gainsight ($10k+/mo) — all too expensive for small SaaS
- Direct gap: Customer.io ($150/mo) and Intercom ($74/mo base) exist as general messaging tools but lack built-in health scoring — they're platforms to build on, not turnkey solutions
