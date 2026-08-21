# Monetization — Optional Architecture (Implemented, Opt-In)

**Status:** Implemented and tested (Phase 26), built on explicit user request
after this phase was deliberately marked "Not Started — explicitly deferred"
in `/docs/ROADMAP.md` and billing/payments was listed as an MVP non-goal in
`/docs/PRD.md` §7. Before building it, the user was asked to explicitly
confirm they wanted monetization architecture built now, and they did.

Billing is **entirely opt-in**. With no `DODO_PAYMENTS_*` environment
variables set (the default for every existing deployment, and for any new
deployment that doesn't set them), every behavior described here is
inactive: usage is unlimited, `GET /api/v1/billing/status` reports
`configured: false`, and `POST /api/v1/billing/checkout` refuses with a 400.
Free Mode and AI Mode with a user's own provider key work identically
whether billing is configured or not.

## 1. Tiers

- **Free** — deterministic analysis, local tooling, audit reports (always,
  regardless of billing configuration). AI Mode works with a monthly cap of
  **50 AI operations** only when billing is configured; unlimited otherwise.
- **Pro** — unlimited AI operations. Activated by a real Dodo Payments
  subscription checkout + webhook-verified `subscription.active` event.

Tier limits live in `backend/src/billing/types.ts`'s `TIER_LIMITS`.

## 2. Payment Provider

**Dodo Payments** (https://dodopayments.com), a merchant-of-record platform.
This module originally targeted Razorpay; Razorpay's merchant onboarding
rejected this product's live account application because self-hosted
software gets auto-classified under their restricted "hosting" business
category — a policy block confirmed by a second, identically-worded
rejection after resubmitting with corrected form answers, not something
fixable by re-describing the business. Dodo Payments explicitly targets
AI/SaaS products, supports individual sellers without incorporation, and has
no equivalent hosting exclusion, so billing was re-pointed at it.

Unlike the original Razorpay integration (a hand-rolled one-time-order +
fixed 30-day period — explicitly scoped that way because real Razorpay
Subscriptions was judged too large an integration at the time), this uses
Dodo's native Subscription product type directly: a checkout session is
created for a subscription product, and `subscription.active` /
`subscription.renewed` webhooks activate/extend the Pro period using Dodo's
own `next_billing_date` when present (falling back to a 30-day period only
if a webhook omits it), while `subscription.cancelled` deactivates
immediately.

Configuration (all via environment variables, all optional — billing stays
off unless all three required ones are set):

- `DODO_PAYMENTS_API_KEY`, `DODO_PAYMENTS_WEBHOOK_KEY`, `DODO_PRODUCT_ID` —
  required together. If any is missing, `loadBillingConfig()` returns `null`
  and every billing check becomes a no-op.
- `DODO_PAYMENTS_ENVIRONMENT` (`test_mode` default, or `live_mode`),
  `DODO_API_BASE_URL` (derived from the environment unless overridden —
  `https://test.dodopayments.com` / `https://live.dodopayments.com`),
  `DODO_RETURN_URL` (default `/settings` — where Dodo's hosted checkout
  redirects the browser after payment; a relative path resolves against the
  server's own request origin) — optional.

## 3. Isolation Principle (as-built)

- All billing/subscription logic lives in `backend/src/billing/` with its
  own DB tables (migration `011_billing.sql`, columns renamed to Dodo's
  naming in `016_dodo_billing.sql`: `subscription`, `billing_webhook_event`).
  Nothing in `analysis/`, `ai/`, `git/`, or `patch/` imports from `billing/`.
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
- `backend/src/billing/dodoClient.ts` — real Dodo Payments Checkout
  Sessions API client (`createCheckoutSession`), Bearer auth, classified
  errors (`auth_error`, `rate_limited`, `unreachable`).
- `backend/src/billing/webhookVerify.ts` — real Standard Webhooks
  (HMAC-SHA256, `whsec_`-prefixed base64 secret, `id.timestamp.payload`
  signed content, timestamp-tolerance replay protection)
  `crypto.timingSafeEqual`-based signature verification.
- `backend/src/billing/subscriptionRepo.ts` — singleton subscription row,
  activation, immediate cancellation, automatic period-expiry downgrade
  (safety net if a cancellation webhook is ever missed), idempotent
  webhook event recording (redelivery-safe).
- `backend/src/billing/usageLimiter.ts` — `checkAiOperationAllowed()`, the
  enforcement function; always unlimited when billing isn't configured.
- `backend/src/routes/billing.ts` — `GET /api/v1/billing/status`,
  `POST /api/v1/billing/checkout`, `POST /api/v1/billing/webhook` (raw-body
  parsing scoped to only this route, for exact-byte signature
  verification).
- `frontend/src/pages/Billing.tsx` (served at `/settings`) — shows
  configured/unconfigured state, current tier, usage vs. limit with a
  visible near-limit/at-limit warning, and an "Upgrade to Pro" flow. Dodo's
  checkout is a real hosted page, not an embedded widget — the browser is
  simply redirected to the `checkout_url` a real server-side session
  returns, then back via `return_url` once payment succeeds or fails. No
  client-side payment script/CSP allow-listing needed at all (unlike the
  earlier Razorpay Checkout widget).

Tests: backend billing-related suites (`dodoClient` behavior exercised via
`routes.billing.test.ts`'s real local-HTTP-server checkout test,
`webhookVerify`, `subscriptionRepo`, `usageLimiter`) plus a full
`routes.billing.test.ts` covering status/checkout/webhook/idempotency/
cancellation and a real end-to-end 402-enforcement test (50 real successful
AI calls, then a real 51st call blocked). Frontend `Billing.test.tsx`
covers the configured/unconfigured/limit states and the real checkout
redirect. Full suites green at time of writing: 458/458 backend, 135/135
frontend.

No live Dodo Payments account/transaction has gone through this code yet —
it's built against Dodo's public API/webhook documentation
(https://docs.dodopayments.com), the same honesty caveat the original
Razorpay client carried before it about not having live credentials in this
development sandbox. Test the flow for real with Dodo's test-mode
credentials (§6 step 5) before switching to live-mode ones.

## 5. Explicit Non-Goals (still true)

- No proration, refunds, or invoice management beyond what Dodo's own
  hosted checkout/customer portal provides.
- No multi-tenant/team billing — single-instance singleton only.
- No pricing page beyond the in-app Settings/Billing status view.

## 6. Setup Guide (turning billing on, step by step)

This section is for a server operator who wants to actually enable the Pro
upgrade flow on their own instance. Nobody needs to do any of this for the
app to work — see the opt-in note at the top of this file. **The app never
asks for these credentials through a form; they're only ever set as
environment variables, on the machine actually running the backend.**

1. **Sign up at [Dodo Payments](https://dodopayments.com)** and complete
   their seller onboarding (as of this writing, they accept individual
   sellers without requiring incorporation — worth confirming current
   policy on their site, since payment-provider onboarding requirements do
   change). Get your API key from the dashboard — this is
   `DODO_PAYMENTS_API_KEY`. Start in **test mode** to try the flow safely
   before switching to live mode.

2. **Create a subscription product** in the Dodo dashboard for the Pro
   tier (set whatever price/billing interval you want — this app doesn't
   hardcode a price, Dodo's product configuration is the source of truth).
   Copy its product id — that's `DODO_PRODUCT_ID`.

3. **Set up a webhook to get `DODO_PAYMENTS_WEBHOOK_KEY`.** In the Dodo
   dashboard, add a webhook endpoint pointed at
   `https://<your-server>/api/v1/billing/webhook` (this server must be
   publicly reachable over HTTPS for Dodo to deliver to it — a plain
   `http://localhost` URL only works if you're tunneling it, e.g. with
   `ngrok`, for local testing). Subscribe at least to `subscription.active`,
   `subscription.renewed`, and `subscription.cancelled` (that's what
   `routes/billing.ts` acts on). Dodo shows you a webhook signing secret
   (formatted `whsec_...`) at creation time — copy it; that's
   `DODO_PAYMENTS_WEBHOOK_KEY`.

4. **Set the environment variables where the backend process will see
   them.** This app does not read a `.env` file automatically unless you
   create one at `backend/.env` — copy `backend/.env.example` to
   `backend/.env` and fill in the `DODO_*` values (the file is gitignored,
   so it's safe to put real secrets in it). `dotenv/config` is imported
   first thing in `src/server.ts`, so anything in `backend/.env` is loaded
   automatically the next time the server starts — no other change needed.
   If you'd rather not use a `.env` file, any of these work exactly as
   well, since `loadBillingConfig()` just reads `process.env` — pick
   whichever fits how you're already running this:
   - **PowerShell (current session only):**
     `$env:DODO_PAYMENTS_API_KEY = "..."` (repeat for the other two),
     then `npm run dev` in the same terminal.
   - **PowerShell (persist across restarts):**
     `[System.Environment]::SetEnvironmentVariable("DODO_PAYMENTS_API_KEY", "...", "User")`
     (repeat for the other two, then open a new terminal).
   - **bash/zsh:** `export DODO_PAYMENTS_API_KEY=...` before starting the
     server, or add the exports to your shell profile / a process
     supervisor's env config.
   - **systemd:** add `Environment=DODO_PAYMENTS_API_KEY=...` lines (or an
     `EnvironmentFile=` pointing at a file in the same `KEY=value` shape as
     `.env.example`) to the unit file.
   - **Docker / docker compose:** add them under `environment:` in
     `deploy/docker-compose.yml`, or pass `--env-file backend/.env` to
     `docker run` / `docker compose up`.

5. **Restart the backend.** `GET /api/v1/billing/status` should now report
   `configured: true`, and the Settings/Billing page in the app will show
   the Free/Pro tier UI and usage instead of "Billing is not configured on
   this server."

6. **Test the flow with test-mode credentials before ever using live-mode
   ones.** Dodo's test mode lets you complete a full checkout without real
   money moving. Only switch `DODO_PAYMENTS_ENVIRONMENT` to `live_mode`
   (and the API key/product id/webhook to their live-mode equivalents) once
   you've confirmed a test-mode checkout activates Pro correctly end to
   end — including that the webhook actually arrives and
   `GET /api/v1/billing/status` flips to `tier: "pro"`.
