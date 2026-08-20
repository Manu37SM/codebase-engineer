# Security Model — Codebase Engineer

## 1. Threat Model

Repositories opened by Codebase Engineer are treated as **untrusted input**.
A repository may contain malicious file names, symlinks, oversized files,
crafted paths designed to escape the project root, or secrets committed by
accident. The application must remain safe to point at an arbitrary repository.

## 2. Path Sandboxing

- Every project is registered with a single `root_path`.
- All filesystem operations resolve the target path with `path.resolve` and
  verify the resolved absolute path is a descendant of `root_path` before any
  read/write/exec. Any path that resolves outside the root (e.g. via `../../`
  traversal or a symlink escape) is rejected with an explicit error — never
  silently clamped.
- Symlinks inside a repository are not followed outside the sandbox boundary
  during indexing.

## 3. Excluded / Ignored Content

The indexer respects `.gitignore` and project-specific ignore rules, and never
scans `.git`, `node_modules`, `target`, `build`, `dist`, generated artifacts, or
large binary files unless explicitly requested by the user for a specific
operation.

## 4. Secret Detection & Redaction

A deterministic pattern-based scanner (`backend/src/security/`) flags likely
secrets: API keys, JWT secrets, passwords in config, private key blocks, cloud
credential patterns, database connection strings with embedded credentials.

Rules:
- A detected secret is **never** stored, logged, or displayed in full.
- Findings referencing a secret show a redacted form only (e.g. first 4 and
  last 2 characters, rest masked) plus file/line location.
- Secrets are stripped before any content is included in an AI context bundle
  (see `/docs/AI_MODE.md` §3) — the context sanitization layer runs before
  network egress to any AI provider, not after.

## 5. AI Egress Control

- No AI request is sent unless the user has explicitly configured a provider
  and explicitly triggered an AI action (explain / root-cause / fix-plan /
  generate patch / diagnose / self-review).
- Only the minimum context selected by the Context Engine is sent — never a
  full repository dump.
- Provider API keys are stored server-side only (backend), referenced from
  `provider_configuration` by an opaque `api_key_ref`, and are never returned
  to the frontend or logged.

## 6. No Autonomous Execution

- AI-generated code changes are surfaced as a diff (`patch.diff_text`) and
  never written to disk automatically. A `patch` only transitions to `applied`
  after an explicit `patch_review` with `decision = approved`.
- AI-generated shell/test commands are never executed automatically. The
  built-in test runner only executes a fixed, allow-listed set of commands
  derived from detected build tooling (e.g. `mvn test`, `npm test`) — it does
  not execute arbitrary AI output.

## 7. Data Storage

- SQLite database is local-only, stored under the application's data directory
  on the user's machine.
- `ai_request` / `ai_response` tables store metadata (provider, model, token
  estimates, operation type, timestamps, status) by default. Raw prompt/response
  bodies are stored only if the user opts in via Settings, and are still passed
  through the redaction layer first.

## 8. Dependency & Supply Chain

- The application itself pins dependencies via lockfiles (`package-lock.json`).
- Dependency analysis of the *target* repository never auto-installs or
  auto-updates the target's dependencies; it only reads manifests/lockfiles.

## 9. Known Gaps (updated Phase 22)

As of Phase 22 (a dedicated hardening/verification pass, not new product
surface — see `/docs/FEATURE.md`'s Platform/Scaffolding table for the full
audit writeup), every control this document describes has a real
implementation, is covered by real tests, and was re-confirmed by reading
the actual enforcement code (not just re-reading this document):

- §2 Path sandboxing: `resolveWithinRoot`/`assertValidProjectRoot`
  (`backend/src/security/paths.ts`, 9 tests). Every route that touches a
  project's filesystem does so via `project.root_path`, which is validated
  once server-side at registration and never re-supplied by the client per
  request. The one route where an AI-*proposed* path reaches disk
  (`POST .../generated-tests/:testId/write-and-run`) explicitly re-validates
  that path through `resolveWithinRoot` before writing, refusing a 400 if it
  escapes the root.
- §3 Excluded/ignored content: default-exclude + nested-`.gitignore`-aware
  file walking (see `docs/ARCHITECTURE.md` for the documented approximation
  nested `.gitignore` merging makes vs. real `git check-ignore`).
- §4 Secret detection & redaction: the `hardcoded-secret`/`env-file-committed`
  analysis rules and finding-evidence redaction (Phase 6/10), plus AI-context
  redaction — every content path that reaches a provider (`select.ts`'s
  Finding-target bundle, `selectContextForTestFailure.ts`'s TestRun-target
  bundle — the only two context builders in the product) routes through
  `redactSecretsInText` before being counted toward the token budget, with
  no bypass found on audit. A generated patch's `diff_text`, passed raw into
  Phase 21 self-review's prompt, is safe by construction: it's the
  provider's own output, generated only from already-redacted context, never
  a fresh unredacted read of the file.
- §5 AI egress control: real as of Phase 14 onward. A provider's API key is
  stored server-side only and never returned in any API response or logged
  (`db/aiProviderRepo.ts`'s `toPublic`/`maskApiKey`). Every AI-Mode action
  (explain / root-cause / fix-plan / generate patch / generate test /
  diagnose / self-review) only sends the Context Engine's bounded, redacted
  selection, and only ever fires on an explicit user-triggered API call —
  never on page load, a timer, or as a side effect of another action.
- §6 No autonomous execution: real as of Phase 17-19. Both routes that write
  to or execute against a project's real filesystem
  (`POST .../patches/:patchId/apply`, `POST .../generated-tests/:testId/write-and-run`)
  check the persisted approval-gate `status` server-side before acting, not
  just in the UI — verified by tests that call each route prematurely and
  confirm a 400 refusal. The test runner (`backend/src/testrunner/detect.ts`)
  only ever executes its own fixed, allow-listed command detection
  (`mvn test`, or the project's own declared `npm`/`pnpm`/`yarn` `test`
  script) — never a command derived from or influenced by AI output.
- §7 Data storage: local-only SQLite; `ai_request`/`ai_response` store
  metadata by default, raw content only if the user opts in (still
  redaction-filtered either way, since it's the same content selected for
  the provider call).
- §8 Dependency & supply chain: `backend/`'s and `frontend/`'s own
  dependencies are pinned via lockfiles; a real `npm audit` pass in Phase 22
  found and fixed genuine high/critical-severity production advisories —
  `fastify` (bumped `^4.28.1 -> ^5.12.1`) and `react-router-dom` (bumped
  `^6.26.1 -> ^7.18.2`) — after confirming this product's actual usage of
  each is limited to long-stable declarative APIs unaffected by either
  major version's breaking changes; both backend (291/291) and frontend
  (79/79) suites reverified stable across repeated runs post-upgrade.
  **Accepted, documented risk**: `npm audit` also flags `vite`/`vitest`/
  `esbuild` (all devDependencies, never shipped in the built application) —
  the underlying advisory is a dev-server CORS/request-forwarding issue that
  only matters if `vite dev`'s dev server is exposed to a network, which
  this project's test tooling never does. Fixing it requires a `vitest`
  `3.x -> 4.x` major bump across every one of the 34 backend + 9 frontend
  test files' APIs — a real regression risk disproportionate to a dev-only,
  network-exposure-contingent issue. Left unfixed deliberately; revisit if
  `vitest` 4.x adoption becomes necessary for another reason. Target-repository
  dependency analysis (Phase 10) never auto-installs/auto-updates the
  *target* repository's own dependencies, only reads manifests/lockfiles —
  unaffected by this section, which is about this product's own dependencies.

## 10. Pre-Launch Checklist Audit (2026-08-20)

The user asked for a direct code audit against a generic 40-item
"things to do before launching your app" checklist (API keys, RLS,
session cookies, rate limiting, CORS, security headers, etc.), re-run
after the auth + import/apply-mode feature arc (Tasks #80-#94). Findings,
verified against the actual code (not assumed from file/function names):

**Already real (confirmed by reading the enforcement code):** secrets are
never hardcoded and never committed (`.env` gitignored, `.env.example`
only); OAuth/password reset tokens are encrypted at rest (§4 above) and
passwords are hashed with `scrypt` at OWASP-recommended parameters, never
stored or logged in plaintext (`auth/password.ts`); server-side auth is a
single, always-applied choke point (`auth/guard.ts`), not a
frontend-only gate; session cookies are `httpOnly` + `sameSite=lax` +
protocol-aware `secure` (`auth/session.ts`); every dynamic SQL query in
`backend/src/db/` uses parameterized placeholders, never string-interpolated
user values (spot-checked `findingRepo.ts`/`fileRepo.ts`'s dynamic
`WHERE` builders specifically, since those are the ones that
template-literal-interpolate a clause — only ever a hardcoded column
name, values always bound via `?`); the frontend never uses
`dangerouslySetInnerHTML` anywhere, relying on React's default escaping
for all user/AI-sourced content; CORS is not just "restricted" but
entirely absent by design — the frontend and API are always same-origin
(the dev proxy in `frontend/vite.config.ts` handles the one case where
they'd otherwise differ), which is a *stronger* posture than an
allowlist; AI-Mode never auto-applies a model's output — every patch
requires an explicit human approval step (`approved_for_apply` status)
before `/apply` will act on it, which is this product's actual defense
against prompt injection (not a content filter, a mandatory human
in the loop); AI usage is capped per Phase 26's billing module when
configured; Razorpay webhooks are HMAC-verified (`billing/webhookVerify.ts`)
and prices/limits are read server-side from `billing/types.ts`'s
`TIER_LIMITS`, never trusted from the client.

**Fixed as part of this audit (both real code changes, not just this
doc):**
- **No security response headers at all** (no CSP, `X-Frame-Options`,
  `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`, or
  HSTS) — added via a plain `onSend` hook (`backend/src/security/headers.ts`),
  not a new dependency. HSTS is only sent when the request is (or is
  proxy-trusted to be) HTTPS, the same protocol-aware pattern the
  session cookie's `secure` flag already used, so it can't break the
  default local-http self-hosting case. Verified with a real running
  server + real `curl` request, not just `app.inject()`.
- **No rate limiting on `/auth/login` or `/auth/register`** — local
  accounts had zero protection against scripted password guessing or
  mass account creation beyond opt-in, not-yet-wired-into-the-frontend
  Turnstile (see below). Added a minimal in-memory fixed-window limiter
  (`backend/src/auth/rateLimit.ts`, no new dependency, consistent with
  this project's dependency-minimalism convention) — 10 attempts per 15
  minutes per IP for login, 10 per hour for registration, keyed by IP
  alone (not IP+email) so an attacker can't dodge it by cycling
  candidate emails. A genuine successful login clears the counter so a
  real user who mistypes a few times isn't then punished.

**Real, currently-open gaps — flagged rather than silently left
undocumented:**
- ~~Turnstile bot verification is backend-only plumbing~~ — **closed.** The
  frontend widget (`frontend/src/components/TurnstileWidget.tsx`,
  `frontend/src/lib/turnstile.ts`) is now wired into both the login and
  register forms, gated on the build-time `VITE_TURNSTILE_SITE_KEY` env
  var (unset = renders nothing, exactly today's behavior). CSP's
  `script-src`/`frame-src`/`connect-src` were extended to allow
  `challenges.cloudflare.com` (`backend/src/security/headers.ts`) and the
  Docker build now threads the site key through as a build ARG (Vite env
  vars are baked in at build time, not read at container runtime — see
  `deploy/Dockerfile` and `deploy/docker-compose.yml`'s `build.args`).
  Still requires the operator's own Cloudflare site/secret key pair,
  which is their call to obtain — nothing here fabricates one.
- **No password-reset feature exists at all** (no forgot-password route,
  no reset-link email). This means "expire reset links," "rate limit
  password resets," and "reset sessions on password change" from the
  checklist are not applicable — there's no such flow to secure yet, not
  a secured-but-broken one.
- **No per-account data isolation on projects/findings/patches.** Only
  the `user`/`session`/`oauth_identity` tables are scoped by `user_id`;
  `project` and everything under it are not. In this product's
  documented architecture (self-hosted, typically single-user or a
  small team sharing one instance, explicitly *not* a multi-tenant
  SaaS), auth is a perimeter control — "is this a legitimate user of
  this instance" — rather than per-user data partitioning within it.
  That's a deliberate scope match to the stated architecture, not an
  oversight, but it means anyone who registers an account on a shared
  instance can see every project registered on it. Worth confirming
  explicitly if the intended deployment is ever a shared multi-user
  instance rather than one person's own machine.
- **No account lockout after repeated failed logins beyond the new IP
  rate limit above** — a determined attacker distributing guesses
  across many IPs isn't slowed by an IP-keyed limiter. Not added in
  this pass since it requires a product decision (lock the account
  entirely? for how long? notify the owner how?) rather than a
  mechanical fix.
- **A registration attempt for an already-used email returns 409**,
  which does let a caller enumerate which emails have accounts (the
  *login* path already avoids this — same error whether the email or
  password is wrong — but *register* does not). Left as-is rather than
  silently changed, since suppressing it would also remove real user
  feedback ("this email is already taken") that most people expect from
  a signup form; flagging as a tradeoff for the user's call, not fixing
  unilaterally.
- **A fresh `npm install` on the user's Windows machine reported 5
  dependency vulnerabilities (3 moderate, 1 high, 1 critical)** in the
  backend's dependency tree, beyond what was resolved in Phase 22's own
  `npm audit` pass. Could not be independently investigated or fixed in
  this session (no network access in the cloud sandbox this audit ran
  in, so no fresh `npm audit`/`npm audit fix` could be run and verified
  against the real test suite). Recommend the user run `npm audit`
  (without `--force` first) on their own machine and share the output
  before applying any fix, since `audit fix --force` can silently pull
  in breaking major-version bumps.
- No admin-only routes exist in this product at all (so "remove default
  admin routes" doesn't apply), and `@fastify/static` never serves a
  directory listing for an unmatched path — it 404s or falls through to
  the SPA — so directory listing was already not exposed.

2 new backend test files (`auth.rateLimit.test.ts`, `security.headers.test.ts`)
plus 3 new tests in `auth.test.ts` covering the rate limiter through the
real `/login`/`/register` routes — 459/459 backend tests total (was 447),
stable across repeated runs; 136/136 frontend tests unaffected; both
builds clean; the security-header change was also verified against a
real running server via real `curl`, not just `app.inject()`.

Do not treat this document as evidence that a control is active until its
corresponding phase is marked "Done (tested)" in `/docs/FEATURE.md`.
