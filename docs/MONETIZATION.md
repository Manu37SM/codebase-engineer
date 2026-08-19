# Monetization — Optional Architecture (Implemented, Opt-In)

**Status:** Implemented and tested (Phase 26), built on explicit user request
after this phase was deliberately marked "Not Started — explicitly deferred"
in `/docs/ROADMAP.md` and billing/payments was listed as an MVP non-goal in
`/docs/PRD.md` §7. Before building it, the user was asked to explicitly
confirm they wanted monetization architecture built now, and they did.

Billing is **entirely opt-in**. With no `RAZORPAY_*` environment variables
set (the default for every existing deployment, and for any new deployment
that doesn't set them), every behavior described here is inactive: usage is
unlimited, `GET /api/v1/billing/status` reports `configured: false`, and
`POST /api/v1/billing/checkout` refuses with a 400. Free Mode and AI Mode
with a user's own provider key work identically whether billing is
configured or not.

## 1. Tiers

- **Free** — deterministic analysis, local tooling, audit reports (always,
  regardless of billing configuration). AI Mode works with a monthly cap of
  **50 AI operations** only when billing is configured; unlimited otherwise.
- **Pro** — unlimited AI operations. Activated by a real Razorpay order +
  webhook-verified payment, for a 30-day period from activation.

Tier limits live in `backend/src/billing/types.ts`'s `TIER_LIMITS`.

## 2. Payment Provider

Razorpay. This is a one-time-order-plus-webhook-verified-payment flow, not a
real Razorpay Subscriptions (recurring billing) integration — each Pro period
is a discrete 30-day grant activated by a single successful payment. Moving
to real recurring subscriptions would be a natural next step if this is ever
used for real; it was not attempted here speculatively, since nothing in
this codebase depends on recurring billing existing.

Configuration (all via environment variables, all optional — billing stays
off unless all three required ones are set):

- `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET` —
  required together. If any is missing, `loadBillingConfig()` returns `null`
  and every billing check becomes a no-op.
- `RAZORPAY_PRO_PRICE_PAISE` (default `999900` = ₹9,999), `RAZORPAY_PRO_PRICE_CURRENCY`
  (default `INR`), `RAZORPAY_API_BASE_URL` (default
  `https://api.razorpay.com/v1`, overridable for testing) — optional.

## 3. Isolation Principle (as-built)

- All billing/subscription logic lives in `backend/src/billing/` with its
  own DB tables (migration `011_billing.sql`: `subscription`,
  `billing_webhook_event`). Nothing in `analysis/`, `ai/`, `git/`, or
  `patch/` imports from `billing/`.
- The billing-agnostic `ai_request` table (`/docs/AI_MODE.md` §7) was not
  changed. `usageLimiter.ts` only reads from it (counting rows in the
  current calendar month) — it never writes to it and required no schema
  change to it.
- Core repository-analysis features (Free Mode) have no runtime dependency
  on the billing module. The only integration point is a single choke-point
  check inside `routes/projects.ts`'s `resolveEnabledProvider()` — the
  shared helper already used by every AI-spending route (explain,
  root-cause, fix-plan, generate-patch, generate-test, diagnose,
  self-review) — which now calls `checkAiOperationAllowed()` first and
  returns a `402` before resolving a provider if the caller is over their
  limit.
- `subscription` is a single-row singleton (this product is single-instance/
  local-first per `/docs/PRD.md` §7 — no multi-tenant/team billing).

## 4. What's Implemented

- `backend/src/billing/config.ts` — env-var-driven config, `null` when
  unconfigured.
- `backend/src/billing/razorpayClient.ts` — real Razorpay Orders API client
  (`createOrder`), HTTP Basic Auth, classified errors (`auth_error`,
  `rate_limited`, `unreachable`).
- `backend/src/billing/webhookVerify.ts` — real HMAC-SHA256 webhook
  signature verification (`crypto.timingSafeEqual`).
- `backend/src/billing/subscriptionRepo.ts` — singleton subscription
  row, activation, automatic period-expiry downgrade, idempotent webhook
  event recording (redelivery-safe).
- `backend/src/billing/usageLimiter.ts` — `checkAiOperationAllowed()`, the
  enforcement function; always unlimited when billing isn't configured.
- `backend/src/routes/billing.ts` — `GET /api/v1/billing/status`,
  `POST /api/v1/billing/checkout`, `POST /api/v1/billing/webhook` (raw-body
  parsing scoped to only this route, for exact-byte signature
  verification).
- `frontend/src/pages/Billing.tsx` (served at `/settings`) — shows
  configured/unconfigured state, current tier, usage vs. limit with a
  visible near-limit/at-limit warning, and an "Upgrade to Pro" flow. If the
  Razorpay Checkout widget script (`window.Razorpay`) isn't loaded on the
  page, this honestly reports that the payment widget can't open rather
  than fabricating success — a real order is still created server-side.

Tests: 27 new backend tests across 4 files (`razorpayClient`,
`webhookVerify`, `subscriptionRepo`, `usageLimiter`) plus a full
`routes.billing.test.ts` covering status/checkout/webhook/idempotency and a
real end-to-end 402-enforcement test (50 real successful AI calls, then a
real 51st call blocked). 7 new frontend tests for the Billing page. Full
suites: 332/332 backend, 86/86 frontend, both stable across repeated runs.

## 5. Explicit Non-Goals (still true)

- No real recurring Razorpay Subscriptions integration (see §2).
- No multi-tenant/team billing — single-instance singleton only.
- No proration, refunds, or invoice management.
- No pricing page beyond the in-app Settings/Billing status view.

## 6. Setup Guide (turning billing on, step by step)

This section is for a server operator who wants to actually enable the Pro
upgrade flow on their own instance. Nobody needs to do any of this for the
app to work — see the opt-in note at the top of this file. **The app never
asks for these credentials through a form; they're only ever set as
environment variables, on the machine actually running the backend.**

1. **Get real Razorpay API keys.** Sign up / log in at
   [dashboard.razorpay.com](https://dashboard.razorpay.com), then go to
   Settings → API Keys and generate a key pair. Start with **Test Mode**
   keys (prefixed `rzp_test_`) to try the flow safely before switching to
   Live Mode keys (`rzp_live_`) — the app doesn't care which mode's keys
   you use, Razorpay routes test-vs-live automatically based on the key
   prefix. This gives you `RAZORPAY_KEY_ID` and `RAZORPAY_KEY_SECRET`.

2. **Set up a webhook to get `RAZORPAY_WEBHOOK_SECRET`.** In the Razorpay
   dashboard, go to Settings → Webhooks → Add New Webhook. Point it at
   `https://<your-server>/api/v1/billing/webhook` (this server must be
   publicly reachable over HTTPS for Razorpay to deliver to it — a plain
   `http://localhost` URL only works if you're tunneling it, e.g. with
   `ngrok`, for local testing). Subscribe at least to the
   `payment.captured` event (that's what `webhookVerify.ts` and
   `subscriptionRepo.ts` act on). Razorpay shows you a webhook secret at
   creation time — copy it; that's `RAZORPAY_WEBHOOK_SECRET`.

3. **Set the three environment variables where the backend process will see
   them.** This app does not read a `.env` file automatically unless you
   create one at `backend/.env` — copy `backend/.env.example` to
   `backend/.env` and fill in the three `RAZORPAY_*` values (the file is
   gitignored, so it's safe to put real secrets in it). `dotenv/config` is
   imported first thing in `src/server.ts`, so anything in `backend/.env`
   is loaded automatically the next time the server starts — no other
   change needed. If you'd rather not use a `.env` file, any of these work
   exactly as well, since `loadBillingConfig()` just reads
   `process.env` — pick whichever fits how you're already running this:
   - **PowerShell (current session only):**
     `$env:RAZORPAY_KEY_ID = "rzp_test_..."` (repeat for the other two),
     then `npm run dev` in the same terminal.
   - **PowerShell (persist across restarts):**
     `[System.Environment]::SetEnvironmentVariable("RAZORPAY_KEY_ID", "rzp_test_...", "User")`
     (repeat for the other two, then open a new terminal).
   - **bash/zsh:** `export RAZORPAY_KEY_ID=rzp_test_...` before starting the
     server, or add the exports to your shell profile / a process
     supervisor's env config.
   - **systemd:** add `Environment=RAZORPAY_KEY_ID=...` lines (or an
     `EnvironmentFile=` pointing at a file in the same `KEY=value` shape as
     `.env.example`) to the unit file.
   - **Docker / docker compose:** add them under `environment:` in
     `deploy/docker-compose.yml`, or pass `--env-file backend/.env` to
     `docker run` / `docker compose up`.

4. **Restart the backend.** `GET /api/v1/billing/status` should now report
   `configured: true`, and the Settings/Billing page in the app will show
   the Free/Pro tier UI and usage instead of "Billing is not configured on
   this server."

5. **Test the flow with Test Mode keys before ever using Live Mode ones.**
   Razorpay's test mode lets you complete a full checkout with their
   documented test card numbers — no real money moves. Only switch
   `RAZORPAY_KEY_ID`/`RAZORPAY_KEY_SECRET` to `rzp_live_...` values (and
   point the webhook at the same URL under live mode) once you've confirmed
   a test-mode checkout activates Pro correctly end to end.
