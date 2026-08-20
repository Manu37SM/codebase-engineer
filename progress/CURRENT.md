# Current Progress

**Phase:** All 26 roadmap phases complete, the earlier post-roadmap
bug-fix + feature pass complete, and now a full authentication +
import/apply-mode feature arc (Tasks #80–#94) complete: local
email/password accounts with sessions, Cloudflare Turnstile bot
protection, Google and GitHub OAuth sign-in, a GitHub repo browser and a
Google Drive zip-file picker for one-click project registration (in
addition to the existing local-path/git-URL/zip-URL sources), generic
git-URL and zip/download-URL registration, multi-project-in-folder
detection, broadened language/test-runner support (Go, .NET, RSpec,
pytest, plus 11 more languages), a per-project apply-mode setting
(direct-to-disk vs. download-as-zip for AI-Mode patches), a collapsible
sidebar, navigation/onboarding UX fixes, and a Workspace rename with a
remove-project action. All of it implemented, tested, and transferred to
the user's Windows checkout, committed to all three git repos (root,
`backend/`, `frontend/`).
**Last updated:** 2026-08-20

## Status
- **Auth + import/apply-mode arc (Tasks #80–#94): complete.** Every task
  in the range implemented, tested, and transferred/committed to the
  Windows checkout; `git status --short` is clean in all three repos
  (root, `backend/`, `frontend/`) as of this update. Full per-feature
  detail is in `docs/FEATURE.md`'s rows for each task; `docs/AUTH.md`
  covers the local-account and both OAuth flows end to end, including
  exactly which scopes are requested and why.
  - **#80–81**: local accounts (email/password, `bcrypt`, session
    cookies) + Cloudflare Turnstile on register/login.
  - **#82–83**: Google and GitHub OAuth sign-in, both optional/independent,
    tokens encrypted at rest (AES-256-GCM).
  - **#84**: GitHub repo browser + clone-to-register, using the
    `repo`-scoped token from GitHub sign-in.
  - **#85**: generic git-URL clone and zip/download-URL import as two
    more registration sources.
  - **#86**: Google Drive zip-file picker — `drive.readonly` scope added
    to Google sign-in, an access-token refresh helper using the stored
    refresh token (Drive browsing can happen well after the ~1 hour
    access-token lifetime), `GET /api/v1/google-drive/zips` +
    `POST /api/v1/google-drive/import`, and a fifth "Google Drive" mode
    in the register-project form.
  - **#87**: multi-project-in-folder detection (reuses the existing
    gitignore-aware file walker to find nested `package.json`/`pom.xml`/
    etc. marker files) + a Repositories-page picker to register any of
    them separately.
  - **#88**: collapsible, icon-only sidebar with a persisted preference.
  - **#89**: broadened language detection (11 more languages) and
    test-runner support (Go, .NET, RSpec, pytest), with real
    process-execution tests against actually-installed `pytest`/`go`.
  - **#90**: per-project apply-mode setting — an approved AI-Mode patch
    either writes straight to the real project ("direct", the default,
    unchanged prior behavior) or is downloadable as a pre-applied zip
    ("download") without ever touching the real files, via
    `buildPatchZip()` reusing the same `applyPatchToDisk()` machinery
    against a scratch temp directory.
  - **#91**: the auth system's routes are actually enforced (a single
    `authGuard` choke point on every request) with real login/register
    UI, not just present-but-unused scaffolding.
  - **#93–94**: navigation/onboarding UX fixes (an empty-state that
    guides a first-time user rather than a blank page) and a rename of
    the old "Dashboard" concept to "Workspace" with a real remove/delete
    project action (workspace-record-only — a repository's actual files
    on disk are never touched by "Remove").
- All work in this arc followed the user's binding architectural
  constraint, restated for each task: self-hosted/local-first only.
  GitHub/Google OAuth are used strictly for authentication and
  repository/Drive access on the *signed-in user's own* behalf — no
  third-party server, no per-user server-side storage beyond this
  single instance's own SQLite database and local disk, and no hosted
  multi-user SaaS redesign anywhere in the arc.
- **Test counts as of this update**: 447/447 backend tests, 136/136
  frontend tests, both suites run clean, both builds (`tsc -b`/
  `vite build`, and the backend's own `tsc -p tsconfig.json` +
  migration/frontend copy steps) clean.

## Notes
- Phase 26 added an entirely opt-in monetization architecture: with no
  `RAZORPAY_KEY_ID`/`RAZORPAY_KEY_SECRET`/`RAZORPAY_WEBHOOK_SECRET` set
  (the default for every existing/new deployment), billing is fully
  inactive — usage unlimited, `/api/v1/billing/status` reports
  `configured: false`. A self-contained `backend/src/billing/` module
  (own DB tables via additive migration `011_billing.sql`, migration-
  safety-verified against a simulated pre-existing database) implements a
  real Razorpay Orders API client, real HMAC-SHA256 webhook signature
  verification, a single-row-singleton subscription record (this product
  is single-instance/local-first), and a usage limiter enforcing a
  50-operations/month free-tier cap only when configured. Enforcement is
  wired into the existing single choke point (`resolveEnabledProvider()`
  in `routes/projects.ts`, already shared by all 7 AI-spending routes) —
  a 402 is returned before any provider is even resolved once the caller
  is over their limit. New `routes/billing.ts` (status/checkout/webhook,
  the webhook route using raw-body parsing scoped to only itself). New
  `frontend/src/pages/Billing.tsx`, now served at `/settings`, with an
  honest degrade path when the Razorpay Checkout widget script isn't
  loaded (a real order is still created; the UI says so rather than
  faking success). Full detail in `docs/MONETIZATION.md` and
  `docs/FEATURE.md`.
- Phase 25 added self-hosting: a real root `Dockerfile` (multi-stage,
  Debian-based for `better-sqlite3` glibc compatibility, installs `git`
  as a real runtime dependency, runs as an unprivileged user) plus
  `docker-compose.yml`, and a systemd user unit
  (`deploy/codebase-engineer.service`) for a non-Docker Linux path. See
  `docs/DEPLOYMENT.md` — note its pre-#80 callout that the product had
  no built-in authentication is now superseded by the #80–#91 auth
  system; a self-hoster exposing the instance beyond their own machine
  should still put it behind their own reverse proxy/TLS as defense in
  depth, but the app itself now gates every route.
- Phase 24 (earlier) added single-process packaging (the backend can
  serve the built frontend) — see `docs/PACKAGING.md`.
- Phase 23 was a measure-first performance pass (~48% faster full scan
  cycle on a real corpus after fixing an O(N²) rule) — see
  `docs/PERFORMANCE.md`.
- Phase 22 was a security-hardening/verification pass (all controls
  confirmed real by code audit, two genuine dependency vulnerabilities
  found and fixed via a real `npm audit`) — see `docs/SECURITY.md` §9.
- Known limitation carried forward, unchanged since Phase 21: import
  extraction is regex-based, not AST-based; only one AI provider adapter
  (`openai-compatible`) is implemented; providers aren't guaranteed to
  follow requested response formats; every AI-Mode advisory workflow
  surfaces the model's own claims for a human to weigh, without
  independently verifying them against the code.
- Backend: 447/447 tests passing, full rebuild clean, stable across
  repeated runs. Frontend: 136/136 tests passing, full rebuild clean,
  `tsc -b` clean, `vite build` clean.
