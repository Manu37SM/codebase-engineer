# Current Progress

**Phase:** All 26 roadmap phases complete. Two pieces of post-roadmap work
are now also in progress at the user's explicit request: (1) a Windows
testrunner timeout bug, round 2 — a real pasted Windows `npm test` run
surfaced a genuine hang the round-1 fix didn't cover, now fixed and
awaiting a third confirmation run; (2) a repo-layout change (per-subproject
git repos + READMEs, deployment files moved into `deploy/`) — implemented,
awaiting the user's confirmation that the moved/new files landed correctly
on their machine and that `git init` at the root went as expected.
**Last updated:** 2026-08-19

## In progress
- **Windows test-runner fix, round 3** (round 2 was insufficient — see
  below): a real pasted Windows `npm test` run showed
  `runTests — real process execution > kills a long-running command on
  timeout and reports timedOut` still failing, hanging until Vitest's own
  unrelated 10s per-test timeout rather than resolving on the function's
  real 300ms `timeoutMs`, plus the same run's `EBUSY: resource busy or
  locked` on temp-directory cleanup. Root cause: even with round 1's
  `detached: false`, `taskkill /pid <pid> /t /f` doesn't reliably make the
  child's `close` event fire on Windows (grandchild processes spawned
  through npm's `.cmd` wrapper can hold duplicate handles to the piped
  stdio streams). Fix: `backend/src/testrunner/run.ts` now applies a
  bounded 2-second grace period after killing the tree, force-resolving
  with `timedOut: true` if `close` still hasn't fired — no behavior change
  on POSIX. `backend/test/fixtures.ts`'s `cleanupRepo()` now uses
  `fs.rmSync`'s built-in `maxRetries`/`retryDelay` to absorb the
  transient Windows file-lock race on cleanup. Verified in the Linux
  sandbox (332/332, twice) and transferred to the user's Windows
  checkout — **still awaiting a third real pasted Windows `npm test` run
  to confirm this round actually closes it out**, since round 1 looked
  sufficient in the sandbox too and wasn't.
- **Repo layout change**, at the user's explicit request (they'd
  independently already set up `backend/` and `frontend/` as separate git
  repos and asked for a root git repo, per-subproject READMEs, and the
  Docker/MD files organized): added `backend/README.md` and
  `frontend/README.md`; moved `Dockerfile`/`docker-compose.yml` from the
  repo root into `deploy/` (build context unchanged — still the repo
  root; compose file updated with `build.context: ..`, `dockerfile:
  deploy/Dockerfile`; docs updated with the new `-f deploy/... --project-
  directory .` invocation); moved `CHANGELOG.md` into `docs/`; added a
  root `.gitignore` excluding `/backend/` and `/frontend/` (each stays
  independently version-controlled, per the user's explicit choice — not
  folded into one history); root `README.md` gained a "Version control
  layout" section explaining the three-separate-repos structure. The
  actual `git init` + first commit at the project root, and the `backend`/
  `frontend` repos' own commits of their pending changes, were run
  directly against the user's real Windows checkout via the device
  bridge's shell access (not just files delivered for the user to run
  manually) — see `docs/CHANGELOG.md`'s "Changed" entry for the full
  list of what moved. Real friction hit and worked around along the way:
  the device bridge's mounted filesystem cannot delete files (`rm`
  reliably fails with `Operation not permitted`even for a stale git
  `index.lock`) — `mv` (rename) works fine, so moves used `mv`, and any
  leftover file that would normally be deleted got moved into a
  `_to_delete/` folder at the project root for the user to remove
  themselves (see the "Notes" callout below for exactly what's in there,
  if anything's left after cleanup).

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
  faking success). 27 new backend tests (332/332 total, including a real
  end-to-end test making 50 real successful AI calls then confirming a
  real 51st is blocked with 402) and 7 new frontend tests (86/86 total).
  Both suites stable across 2 repeated runs; both builds (`tsc -b`/
  `vite build`) clean. Full detail in `docs/MONETIZATION.md` (rewritten
  from "not implemented" to reflect what's actually built) and
  `docs/FEATURE.md`.
- Phase 25 added self-hosting: a real root `Dockerfile` (multi-stage,
  Debian-based for `better-sqlite3` glibc compatibility, installs `git`
  as a real runtime dependency, runs as an unprivileged user) plus
  `docker-compose.yml`, and a systemd user unit
  (`deploy/codebase-engineer.service`) for a non-Docker Linux path. Both
  were actually exercised, not just written: the Docker path was built
  and run end to end (a real container, real curl requests, a real
  bind-mounted repo registered and analyzed through the real API
  producing 61 real findings, and persistence confirmed across a full
  container recreation via a named volume); the systemd path was
  verified with `systemd-analyze verify`, which caught and led to fixing
  a genuine portability bug (a hardcoded Node path) but could not be
  verified as a live-running service in this sandbox (documented as
  such, not glossed over). New `docs/DEPLOYMENT.md` covers both paths in
  full, plus an explicit, unambiguous callout that this product has no
  built-in authentication — a self-hoster exposing it beyond their own
  machine needs to add their own reverse proxy + auth.
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
- Backend: 332/332 tests passing, full rebuild clean, stable across
  repeated runs. Frontend: 86/86 tests passing, full rebuild clean,
  `tsc -b` clean, `vite build` clean.
