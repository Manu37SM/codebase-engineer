# Changelog

## Unreleased

### Added
- Phase 0 documentation set (`docs/PRD.md`, `ARCHITECTURE.md`, `FEATURE.md`,
  `ROADMAP.md`, `SECURITY.md`, `TESTING.md`, `ADR.md`, `AI_MODE.md`,
  `MONETIZATION.md`) and progress tracking (`progress/`).
- Phase 1 project scaffold: backend (Fastify + TypeScript + SQLite) and
  frontend (React + TypeScript + Vite + Tailwind) with smoke tests.
- Phase 2 repository discovery: path sandboxing, gitignore-aware file walker,
  language/build-system/package-manager/framework/Git detectors, project
  registration + discovery API, 21 backend tests.
- Phase 3 repository indexing: nested-`.gitignore`-aware walking, per-file
  language/LOC/size/hash/test/generated classification, regex-based JS/TS/
  Java import extraction, index + file-listing API, 30 backend tests.
- Phase 4 dashboard: frontend API client, persisted project selection,
  Repositories page (register/list/select/scan), Dashboard page (real
  repository summary), 12 frontend tests.
- Phase 5 architecture explorer: JS/TS + Java import resolution, directory-
  depth module aggregation, weighted dependency edges, external-dependency
  summary, table-based Architecture page with depth control, 44 backend +
  16 frontend tests.
- Phase 6/7 deterministic analysis engine + findings system: 5 rules
  (large-file, large-function, todo-fixme-density, missing-test-file,
  hardcoded-secret) producing evidence-backed `Finding` records with
  redacted secrets, `missing-test-file` using BFS transitive import-graph
  closure to avoid false positives, analysis-run persistence, analysis +
  findings API, Findings page with severity/category filters, 63 backend +
  22 frontend tests.
- Phase 8 git analysis: recent commit log, file churn (commits-per-file
  within a configurable window), uncommitted-changes diff stat vs HEAD, all
  computed live off `.git` (not persisted), `GET /projects/:id/git` API, a
  new "Git activity" section on the Dashboard page, 77 backend + 25 frontend
  tests.
- Phase 9 test runner: Maven/npm/pnpm/yarn test-command detection, real
  subprocess execution via `spawn` with process-group timeout kill, Vitest
  and Maven Surefire output parsing, `test_run` persistence (migration 003),
  tests run + history + single-run API, Tests page, 100 backend + 31
  frontend tests.
- Phase 10 dependency + security analysis: npm (lockfile v2/v3 duplicate-
  version detection) and Maven (regex `pom.xml` parsing) dependency
  analysis computed live, `GET /projects/:id/dependencies` API, Dashboard
  Dependencies section; 3 new deterministic security rules
  (`env-file-committed`, `permissive-cors`, `disabled-tls-verification`)
  alongside a new live security-scan endpoint (`GET /projects/:id/security`,
  runs only the security-category rule subset fresh off disk); 122 backend
  + 33 frontend tests.

- Phase 11 audit report: `GET /projects/:id/audit` aggregates existing
  persisted (repository snapshot, latest analysis-run counts, latest test
  run) and live-computed (security scan, dependency analysis, Git
  activity) data into one consolidated view; `GET /projects/:id/audit/export`
  renders the same aggregate as a downloadable Markdown document. Audit
  page (replacing the Phase 0 placeholder) — first frontend surface for the
  Phase 10 live security-scan endpoint. 129 backend + 39 frontend tests.

### Fixed
- Dashboard test "prompts to scan when the selected repository has no
  snapshot yet" began racing once a third independent concurrent fetch
  (dependencies) was added to the page's effect — switched from a bare
  `findByText` to the `waitFor`+`getByText` pattern already used elsewhere
  in the file for this same race.
- `dependencies.test.ts`'s Maven duplicates-note assertion had a stale
  regex (`/not available for Maven/`) that didn't match the actual message
  wording (`isn't available for Maven`).
- Audit Markdown export was missing a blank line between the "Security
  scan" and "Dependencies" sections when there were zero security findings.

## Unreleased (continued)

### Added
- Phase 12 AI provider abstraction: `AIProvider` interface + one real
  adapter (`openai-compatible`, covering OpenAI and OpenAI-compatible local
  servers like Ollama/LM Studio/vLLM) with real HTTP calls, timeout
  handling, and 401/429/network-failure classification. `provider_configuration`
  persistence (migration 004 adds the `api_key` column; the raw key is
  never returned by any API response, only a masked preview). API:
  `GET/POST/PATCH/DELETE /api/v1/ai/providers[/:id]`, `POST .../check-status`,
  `GET .../models`. AI Mode page (replacing the Phase 0 placeholder) for
  configuring and testing providers — explicitly scaffolding, no AI-Mode
  feature calls `complete()` from a real workflow yet. 161 backend + 47
  frontend tests.

## Unreleased (continued)

### Added
- Phase 13 AI context selection engine: `selectContextForFinding()` builds
  a bounded `ContextBundle` for a `Finding` target per `docs/AI_MODE.md`
  §3's 6-step selection order (directly affected file, line-windowed
  method/function approximation, imported symbols, known callers,
  associated test file, relevant config, relevant Git diff hunk), with
  secrets redacted before anything is counted or returned and every
  included/excluded item carrying an honest, real-token-costed reason.
  New shared modules `backend/src/security/secretPatterns.ts` and
  `backend/src/ai/tokenEstimate.ts` (both factored out of existing Phase
  6/12 code, zero regression). API:
  `GET /api/v1/projects/:id/findings/:findingId/context`. Frontend:
  "Preview AI context" toggle on the Findings page. Deliberately scoped
  to `Finding` targets only. 188 backend + 48 frontend tests.

### Fixed
- Context selector: a caller file that was also a test file was being
  added to the candidate list twice, silently shadowing the more specific
  "Test file that imports X" reason with the generic "known caller" one.

## Unreleased (continued)

### Added
- Phase 14 AI finding explanation: `explainFinding()` workflow, the first
  real call to `AIProvider.complete()`, combining Phase 12's provider and
  Phase 13's context bundle (now with an opt-in `includeContent` option)
  into a real prompt. Migration 005 extends `ai_request`/`ai_response`
  with `finding_id`/`content` for accounting and re-display. API:
  `POST /api/v1/projects/:id/findings/:findingId/explain`,
  `GET .../explanation`. Frontend: "AI explanation" toggle on the
  Findings page, visibly disabled with an explanation when no AI provider
  is enabled. Read-only — never writes to a finding or applies anything.
  198 backend + 51 frontend tests.

## Unreleased (continued)

### Added
- Phase 15 AI root-cause analysis: `analyzeRootCause()` workflow, sharing
  context-selection/provider-call/accounting plumbing with Phase 14's
  `explainFinding()` via a new `runFindingWorkflow()` helper. Asks the
  provider for a structured EVIDENCE:/INFERENCE:/CONFIDENCE: response and
  parses it leniently and honestly (unparsed fields left `null`, raw
  response always preserved). No new migration — reuses the Phase 14
  `ai_request`/`ai_response` tables via `operation_type`. API:
  `POST /api/v1/projects/:id/findings/:findingId/root-cause`,
  `GET .../root-cause`. Frontend: "Root-cause analysis" toggle on the
  Findings page, evidence/inference kept visually distinct. 207 backend +
  54 frontend tests.

### Fixed
- Root-cause response parser: the EVIDENCE section regex didn't stop
  before an absent INFERENCE: header, so a response missing that section
  had its CONFIDENCE: line swallowed into the evidence bullets.

## Unreleased (continued)

### Added
- Phase 16 AI fix planning: `planFix()` workflow producing the
  seven-section fix plan docs/AI_MODE.md §5 requires. First workflow to
  fold a prior workflow's output (Phase 15's root-cause analysis) into
  its own prompt as grounding when one exists. New shared
  `parseStructuredSections()` generalizes Phase 15's structured-response
  parser to any number of headers; `rootCauseAnalysis.ts` refactored onto
  it. API: `POST /api/v1/projects/:id/findings/:findingId/fix-plan`,
  `GET .../fix-plan`. Frontend: "Fix plan" toggle on the Findings page.
  Strictly advisory — never a diff, never applied to disk. 224 backend +
  57 frontend tests.

## Unreleased (continued)

### Added
- Phase 17 AI patch generation: `generatePatch()` workflow, folding the
  finding's latest Phase 16 fix plan into the prompt as grounding when
  one exists (`usedFixPlan`). First AI-Mode phase with a real persisted
  state machine (`patch`/`patch_review`): `pending_approval -> approved
  -> proposed`, or `pending_approval -> rejected`, with both approval
  gates enforced server-side against `patch.status` — not just in the UI.
  Migration 006 makes `patch.diff_text` nullable. API:
  `POST/GET /api/v1/projects/:id/findings/:findingId/patches`,
  `GET /api/v1/projects/:id/patches/:patchId`, `POST .../approve`,
  `POST .../reject`, `POST .../generate`. Frontend: "Patches" toggle on
  the Findings page with the full create/approve/reject/generate
  lifecycle UI. The generated diff is stored raw — never parsed,
  validated, or written to disk; applying it is out of scope until
  Phase 18. 235 backend + 62 frontend tests.

### Fixed
- Migration runner: `runMigrations()` ran each migration's rebuild-style
  SQL with `foreign_keys=ON`, so a table-rebuild migration's `DROP TABLE`
  could cascade-delete rows in a dependent table before the rename
  completed (caught by migration-safety verification on migration 006,
  which rebuilds `patch` and would have cascade-deleted real
  `patch_review` rows). Fixed by toggling `PRAGMA foreign_keys` OFF around
  the whole batch of pending migrations and back ON afterward.

## Unreleased (continued)

### Added
- Phase 18 diff review & approval: the second and final human-approval
  gate, and the first phase in the product to write to a file on disk.
  Extends the patch state machine: `proposed -> approved_for_apply ->
  applied`, or `proposed -> rejected`. New `applyPatchToDisk()` always
  runs a real `git apply --check` dry run before ever writing, piping the
  diff via stdin with `execFileSync` (argv array, no shell string).
  Migration 007 adds `patch.apply_error` for recording why an apply
  failed. API: `POST /api/v1/projects/:id/patches/:patchId/approve-apply`,
  `POST .../reject-apply`, `POST .../apply` (retryable from `failed`
  without re-approving). Frontend: the Findings page's "Patches" panel
  gained diff-review/apply/retry actions. 245 backend + 66 frontend
  tests.

## Unreleased (continued)

### Added
- Phase 19 AI test generation: implements docs/AI_MODE.md §1's
  "AI-generated tests (reviewed & executed, not trusted on compile
  alone)". New `generated_test`/`generated_test_review` tables
  (migration 008, purely additive) mirror the patch state machine's
  two-gate shape: `pending_approval -> approved -> proposed ->
  approved_for_write -> written | passed | failed_tests`, or
  `-> rejected`. Only ever creates a NEW file (refuses to overwrite an
  existing path). `generateTest()` folds the finding's fix plan's
  "Required tests" section into the prompt as grounding when one exists.
  The defining step: `/write-and-run` writes the file AND immediately
  runs the project's real, existing test command (Phase 9), persisting a
  real `test_run` row — nothing here is trusted on compile alone. API:
  `POST/GET /api/v1/projects/:id/findings/:findingId/generated-tests`,
  `GET .../generated-tests/:testId`, `POST .../approve`,
  `POST .../reject`, `POST .../generate`, `POST .../approve-write`,
  `POST .../reject-write`, `POST .../write-and-run`. Frontend: a
  "Generated tests" panel on the Findings page mirroring "Patches"'
  shape. 261 backend + 72 frontend tests.

### Fixed
- `backend`'s `npm run build` failed on Windows with "The syntax of the
  command is incorrect.": `copy:migrations` used a POSIX-only shell
  pipeline (`mkdir -p ... && cp *.sql ...`) that only works under
  bash/sh, not npm's default Windows shell (cmd.exe). This had been
  masked throughout development because this session's own environment
  is Linux — caught only once the user built on their actual Windows
  machine. Replaced with `backend/scripts/copy-migrations.mjs`, a plain
  Node.js script using `fs`, which behaves identically on every platform
  this product supports. Verified the fix rebuilds cleanly in this
  session's Linux environment too (not just "removed the Windows-specific
  part") and that all 8 migrations still copy and apply correctly.

## Unreleased (continued)

### Fixed
- Three further real Windows-only bugs, all reported directly by the user
  running `npm test` on their actual Windows machine (7 backend + 1
  frontend failure) — the first real cross-platform verification pass
  this project has had, and each of these was invisible to this session's
  Linux-only development and dogfooding:
  - `applyPatch.ts`'s real `git apply` calls silently rewrote `\n` to
    `\r\n` in written file content, because they inherited the user's
    global `core.autocrlf=true` git config (the common Git-for-Windows
    installer default) — confirmed to be a global setting, not a
    per-repo one, since one of the two failing tests never called
    `initGit()` at all. Fixed by passing `-c core.autocrlf=false -c
    core.safecrlf=false` as config overrides on both the dry-run
    (`git apply --check`) and the real apply, so applied content now
    matches the diff byte-for-byte regardless of the host's git config.
  - `testrunner/run.ts`'s `runTests()` used `node:child_process`'s bare
    `spawn` to invoke `npm`/`pnpm`/`yarn`, which are `.cmd`-wrapped
    batch files on Windows that `spawn` cannot launch without
    `shell: true` — the failure surfaced as `exitCode: null` (the
    process never actually started). Fixed by switching to the
    `cross-spawn` package, which resolves `.cmd`/`.bat` commands
    correctly and internally quotes each argument itself (not a shell
    string built by this code), preserving the "argv array only, never
    a shell string" invocation convention on every platform.
  - The same function's timeout-kill logic used
    `process.kill(-child.pid, "SIGTERM")`, a POSIX process-group kill
    that has no Windows equivalent — Windows doesn't create process
    groups from `detached: true` the way POSIX does, so a timed-out
    Windows test run's child process (and any grandchild test binary it
    spawned) was never actually killed. Fixed by adding a
    `process.platform === "win32"` branch that shells out to
    `taskkill /pid <pid> /t /f` (itself invoked via `cross-spawn` with
    an argv array) to kill the real process tree Windows does track.
  - Frontend: `Dashboard.test.tsx`'s "renders a full dashboard summary
    from a snapshot" test asserted `findByText("my-app")` once and then
    several `getByText(...)` calls synchronously afterward, assuming
    that render was final. The page's initial load resolves through two
    render-apart state updates (a `.then` that sets `snapshot`/
    `fileTotals`, then a separate `.finally` microtask that flips
    `loading` false) on top of `ProjectContext`'s own async project-list
    load — a real race that a slower or differently-scheduled event
    loop (observed on Windows) could lose, leaving the later synchronous
    assertions checking a DOM snapshot that had already moved past.
    Fixed by wrapping the whole assertion group in a single `waitFor`,
    so it only passes once every field is present together in the same
    settled render — this removes the race outright rather than papering
    over one specific timing window.
  - Verified: full backend suite (261/261) and frontend suite (72/72)
    pass in this session's Linux environment, stable across repeated
    runs, and both `backend`'s and `frontend`'s production builds are
    clean. The CRLF and Windows-`taskkill` code paths cannot be executed
    or truly verified outside a real Windows machine, so — consistent
    with the `copy:migrations` fix above — this is verified as "correct
    by inspection and doesn't regress Linux," with real Windows
    confirmation still pending the user's next `npm test` run there.

## Unreleased (continued)

### Added
- Phase 20 AI failure diagnosis: implements docs/AI_MODE.md §4's "(if
  failure) AI Diagnosis" workflow step, the first AI-Mode feature
  targeting a failed `TestRun` instead of a `Finding`. New
  `selectContextForTestFailure()` grounds a context bundle entirely in
  the failed run's own real captured stdout/stderr — the output excerpt
  is always the primary item (truncated-to-fit rather than ever
  excluded), plus real indexed files whose path text is actually found
  inside that output (test-files-first, then by mention count), plus the
  top-matched failing test file's own imports. `diagnoseFailure()` asks
  the provider for `LIKELY_CAUSE:`/`EVIDENCE:`/`SUGGESTED_DIRECTION:` —
  deliberately no diff or code, just a hypothesis and its evidence; an
  actual fix still goes through the existing fix-plan → patch-generation
  → diff-review flow. Migration 009 adds `ai_request.test_run_id`
  (nullable, purely additive, anticipated by migration 005's original
  comment). API: `POST /api/v1/projects/:id/tests/:runId/diagnose`
  (only callable when the run's status is `failed`), `GET
  .../diagnosis`. Frontend: a "Diagnose failure" action on the Tests
  page, shown only for a failed run. 278 backend + 76 frontend tests.
  Dogfooded against a real scratch copy of this project's own backend —
  a real injected bug, a real `vitest` failure, a real diagnosis against
  a real local AI-provider double.

## Unreleased (continued)

### Added
- Phase 21 AI self-review: implements docs/AI_MODE.md §6's self-review
  checklist (correctness, scope creep, regressions, security, missing
  tests, unnecessary complexity, architecture consistency) run against a
  real proposed patch's real diff. Purely advisory — never changes the
  patch's status, never a precondition for approve-apply/apply; can be
  requested any time the patch has a real diff. `selfReviewPatch()`
  reuses the existing `runFindingWorkflow` shared plumbing (now with an
  optional `patchId` passthrough) and asks for all seven checks via the
  shared structured-response parser. Migration 010 adds
  `ai_request.patch_id` (nullable, purely additive) so a self-review's
  accounting stays scoped to the exact diff reviewed, not just the
  finding. API: `POST /api/v1/projects/:id/patches/:patchId/self-review`
  (400 without a diff yet), `GET .../self-review`. Frontend: an "AI
  self-review" toggle inside each patch's card once it has a diff,
  showing a colored pass/concern/fail chip and note per check. 291
  backend + 79 frontend tests. Dogfooded against a real scratch copy of
  this project's own backend — a real hardcoded-secret finding, a real
  fix plan, a real generated diff, and a real self-review against a real
  local AI-provider double, with the patch's own status confirmed
  untouched throughout.

## Unreleased (continued)

### Fixed
- Windows test runner, second correction: the earlier `cross-spawn` +
  `taskkill` fix (below) was insufficient by itself. Real Windows
  `npm test` output showed captured `stdout_ref` coming back empty on
  passing/failing runs, and two "real process execution" tests timing
  out followed by `EBUSY: resource busy or locked, rmdir` errors during
  fixture cleanup - evidence of a child process still alive and holding
  a file lock after the test's own timeout fired. Root cause:
  `runTests()` was passing `detached: true` unconditionally alongside
  piped stdio (`stdio: ["ignore","pipe","pipe"]`); on Windows, where the
  resolved command must go through a `.cmd`/shell wrapper (which
  `cross-spawn` does internally for `npm`/`pnpm`/`yarn`), that
  combination is a known Node.js footgun that can break stdio capture
  and prevent the child from ever emitting `close`. `detached` never
  bought anything on Windows in the first place - `killProcessTree()`'s
  Windows branch already kills the whole tree via `taskkill /pid <pid>
  /t /f`, which walks the OS's own tracked process lineage and doesn't
  need POSIX process-group semantics. Fixed by making it conditional:
  `detached: process.platform !== "win32"`. POSIX behavior (where
  `detached: true` is required for the negative-PID group-kill on
  timeout) is unchanged. Verified: full backend suite (291/291) stable
  across 2 repeated runs and a clean rebuild in this session's Linux
  environment; as with the prior Windows fixes, the Windows-specific
  code path itself cannot be executed here, so real confirmation from
  the user's next Windows `npm test` run is still required before this
  is considered actually fixed - a lesson underscored by this exact fix
  attempt being a correction of one that looked fixed but wasn't.

## Unreleased (continued)

### Added
- Phase 22 security hardening: a systematic verification pass over
  `docs/SECURITY.md`'s controls against the real implementation (not new
  product surface). Audited and confirmed, by reading the actual
  enforcement code: path sandboxing (every route reads a project's
  filesystem only via its server-side-validated `root_path`; the one
  AI-proposed-path write route re-validates via `resolveWithinRoot`
  before writing), AI-context secret redaction (both context builders in
  the product route all content through `redactSecretsInText` before
  it's ever counted toward a token budget, no bypass found), and
  no-autonomous-execution (both disk-writing/executing routes check the
  persisted approval-gate status server-side, and the test runner only
  ever executes its own fixed allow-listed command detection, never
  anything AI-supplied). See `docs/FEATURE.md`'s Platform/Scaffolding
  table for the full writeup.

### Fixed
- Dependency vulnerabilities (found via a real `npm audit`, not assumed):
  `backend`'s `fastify` had real high/critical-severity advisories only
  fixed by the 4→5 major line — upgraded to `fastify@5.12.1` after
  confirming this product's actual usage (a handful of
  `app.get/post/patch/delete` + `request.body/params/query` +
  `reply.status` calls) is unaffected by the major version's breaking
  changes. `frontend`'s `react-router-dom` had a real moderate-severity
  open-redirect advisory only fixed by the 6→7 major line — upgraded to
  `react-router-dom@7.18.2` after confirming usage (`BrowserRouter`/
  `Routes`/`Route`/`Link`/`NavLink`/`Outlet`, all stable across the
  bump) is unaffected. Full backend (291/291) and frontend (79/79)
  suites reverified stable across 2 repeated runs each, with clean
  rebuilds, both before and after each upgrade. Remaining `npm audit`
  findings on both (`vite`/`vitest`/`esbuild`, devDependency-only) left
  as a documented, accepted risk rather than forcing a `vitest` 3→4
  major bump across 34+9 test files for a dev-server-only,
  network-exposure-contingent issue — see `docs/SECURITY.md` §9.
- Windows test runner, second correction (carried over from the prior
  Windows verification loop): `runTests()` in
  `backend/src/testrunner/run.ts` was passing `detached: true`
  unconditionally alongside piped stdio, which broke stdio capture and
  process closure on Windows when the command went through a `.cmd`
  wrapper (real Windows `npm test` output showed empty `stdout_ref` and
  two timed-out tests with subsequent `EBUSY` file-lock errors). Fixed by
  making it `detached: process.platform !== "win32"` — Windows never
  needed it for tree-killing, since `killProcessTree()`'s Windows branch
  already uses `taskkill /pid <pid> /t /f` against the OS's own tracked
  process tree. POSIX behavior (which does need `detached: true` for its
  negative-PID group-kill) is unchanged. Verified stable (291/291, 2x)
  in the Linux sandbox and transferred to the user's Windows checkout;
  not yet independently reconfirmed via a real pasted Windows `npm test`
  run — see `progress/CURRENT.md`.

## Unreleased (continued)

### Added
- Phase 23 performance optimization: a measure-first profiling pass over
  the real discover→index→analyze pipeline against a real 3,672-file
  corpus, plus a real regression-guard test
  (`backend/test/analysis.test.ts`) that builds a synthetic worst-case
  repo and asserts the fixed rule stays well under a bound the old
  algorithm would have blown through — catches any future reintroduction
  of the same class of bug. New `docs/PERFORMANCE.md` records the full
  methodology, data, and what was profiled-and-left-alone because it
  wasn't actually a problem.

### Fixed
- `missing-test-file` analysis rule
  (`backend/src/analysis/rules/missingTests.ts`): the naming-convention
  check re-scanned every path in the repo, from scratch, for every
  candidate source file — an O(files × allPaths) algorithm that measured
  as ~1.4s of a ~3.6s full scan cycle on a real 3,672-file corpus, over
  half of `runAnalysis`'s total time. Fixed by precomputing the set of
  naming-convention-covered base names once in a single O(allPaths) pass
  instead of re-scanning per candidate — turning each check into an O(1)
  lookup. Identical matching semantics (verified: unchanged real finding
  counts on the profiling corpus, all pre-existing correctness tests
  passing unmodified). Result: ~30x faster for this rule, ~2.6x faster
  for `runAnalysis`, ~48% faster for the full scan cycle, on the same
  corpus. Full backend suite (292/292, 1 new test) reverified stable
  across 2 repeated runs with a clean rebuild.

## Unreleased (continued)

### Added
- Phase 24 packaging: the backend can now serve the built frontend and
  run as a single process on a single port, instead of requiring the
  separate dev-time Fastify API + Vite dev server split. `buildApp()`
  (`backend/src/app.ts`) gained an optional `staticDir` — when set to a
  real built-frontend directory, registers `@fastify/static` plus an
  SPA-fallback 404 handler (a direct load/refresh of a client-side route
  like `/findings` correctly re-serves `index.html`; an unmatched
  `/api/*` route still returns a real JSON 404, never silently masked by
  the SPA fallback). New `backend/scripts/copy-frontend.mjs` copies
  `frontend/dist` into `backend/dist/public` as part of `backend`'s own
  `npm run build`, degrading safely (a log message, not a build failure)
  when the frontend hasn't been built yet. `config.ts`'s new `staticDir`
  resolution defaults relative to the compiled module's own location,
  overridable via `CODEBASE_ENGINEER_STATIC_DIR`. Deliberately scoped:
  no installer/desktop-shell (Electron/Tauri)/Docker/hosted-deployment
  decision was made here — that's explicitly deferred to Phase 25
  ("Deployment / self-hosting"); see `docs/PACKAGING.md` for the full
  scope writeup. New `@fastify/static@10.1.3` dependency, confirmed via
  `npm audit` to introduce no new vulnerabilities. 8 new backend tests
  (`staticServing.test.ts`) using real `app.inject()` requests through
  the real Fastify + `@fastify/static` pipeline, including a genuine
  path-traversal check — 300 backend tests total. Dogfooded fully end to
  end: a real running combined process, started from the real built
  artifacts, exercised with real `curl` requests. `README.md`'s
  long-stale "Phase 0/1, no features implemented" status line corrected,
  and a single-process quick-start section added.

## Unreleased (continued)

### Added
- Phase 25 deployment/self-hosting: two real, verified ways to run
  Codebase Engineer outside a dev setup, on top of Phase 24's
  single-process packaging. A multi-stage root `Dockerfile`
  (Debian-based, not Alpine, for `better-sqlite3` glibc compatibility
  across build stages) builds the frontend then the backend (reusing
  Phase 24's `copy-frontend.mjs` unmodified), installs `git` as a real
  runtime dependency, and runs as an unprivileged user; a
  `docker-compose.yml` quick-start. A systemd **user** unit
  (`deploy/codebase-engineer.service`) for running the packaged process
  directly on a Linux host without Docker. New `docs/DEPLOYMENT.md`
  covers both paths plus an explicit security callout: this product has
  no built-in authentication on its own API surface, so self-hosting
  beyond your own machine needs your own reverse proxy + auth. No
  application code changed — pure deployment tooling and docs; 300
  backend / 79 frontend tests unaffected and reverified stable.

### Verified
- Docker path built and run end to end in this project's own dev
  sandbox: a real image built, a real container started and served both
  the API and UI, real `curl` requests confirmed health/UI/`git`
  availability, a real bind-mounted host directory was registered as a
  project through the real API and a real discover→index→analyze cycle
  produced 61 real findings, and the registered project's data survived
  a full container destroy-and-recreate against the same named volume
  (not just a process restart).
- systemd unit verified with a real `systemd-analyze verify` pass, which
  caught a genuine bug in the first draft (a hardcoded `/usr/bin/node`
  path not valid on every install) — fixed to resolve Node via `PATH`
  instead, then reverified clean. Could not verify a live-running
  service in this sandbox (no user D-Bus session available in a cloud
  dev container) — documented honestly as verified-by-static-analysis,
  not verified-by-live-run.

## Unreleased (continued)

### Added
- Phase 26 optional monetization architecture (Razorpay), built only
  after explicit user confirmation (`docs/ROADMAP.md` had marked this
  phase "Not Started — explicitly deferred"; `docs/PRD.md` §7 lists
  billing/payments as an MVP non-goal). Entirely opt-in: with no
  `RAZORPAY_KEY_ID`/`RAZORPAY_KEY_SECRET`/`RAZORPAY_WEBHOOK_SECRET`
  configured (the default), billing is fully inactive and Free Mode/
  AI-Mode-with-a-user's-own-key are unaffected. New self-contained
  `backend/src/billing/` module (own DB tables via additive migration
  `011_billing.sql`, migration-safety-verified) — real Razorpay Orders API
  client, real HMAC-SHA256 webhook signature verification, a single-row
  subscription singleton, and a usage limiter enforcing a 50-operation/
  month free-tier cap only when configured — reading, never writing, the
  existing billing-agnostic `ai_request` table, and imported by nothing in
  `analysis/`, `ai/`, `git/`, or `patch/`. Enforcement wired into the
  single existing choke point (`resolveEnabledProvider()` in
  `routes/projects.ts`, shared by all 7 AI-spending routes), returning a
  402 before resolving a provider once over the limit. New
  `routes/billing.ts` (status/checkout/webhook — a one-time-order-plus-
  webhook-verified-payment flow granting a 30-day Pro period, not real
  recurring Subscriptions). New `frontend/src/pages/Billing.tsx`, now
  served at `/settings`, with an honest degrade path when the Razorpay
  Checkout widget script isn't loaded on the page. 27 new backend tests
  (332 total) including a real end-to-end 402-enforcement test (50 real
  successful AI calls, then a real 51st blocked); 7 new frontend tests
  (86 total). Both suites stable across 2 repeated runs; both builds
  clean. `docs/MONETIZATION.md` rewritten to document what's actually
  built.

## Unreleased (continued)

### Fixed
- Windows test-runner timeout hang, round 2, found by a real pasted
  Windows `npm test` run: after `taskkill /pid <pid> /t /f` kills the
  child process tree on timeout, `close` isn't guaranteed to fire on
  Windows (grandchild processes spawned through `npm`'s `.cmd` wrapper
  can hold duplicate handles to the piped stdout/stderr streams),
  hanging `runTests()` past its own `timeoutMs` — observed as the whole
  test hanging until Vitest's unrelated 10s per-test timeout rather than
  the function's real 300ms budget. `backend/src/testrunner/run.ts` now
  applies a bounded 2-second grace period after killing the tree; if
  `close` still hasn't fired by then, it force-resolves with
  `timedOut: true` rather than waiting on an OS event that may never
  come. No behavior change on POSIX, where `close` already fires
  promptly. Also fixed the same run's `EBUSY: resource busy or locked`
  on temp-directory cleanup (a lingering Windows file lock briefly
  outliving a just-killed process) by giving `cleanupRepo()`
  (`backend/test/fixtures.ts`) `fs.rmSync`'s built-in retry
  (`maxRetries`/`retryDelay`) instead of failing on that timing race.

### Changed
- Repo layout, at the user's explicit request: `backend/` and
  `frontend/` are each intentionally tracked in their own separate git
  repository (not the project root's), and a new root-level git
  repository was set up to cover everything else (`docs/`, `progress/`,
  `deploy/`, this file's own location, and the root `README.md`) — the
  root repo's `.gitignore` excludes `backend/` and `frontend/` outright.
  Added `backend/README.md` and `frontend/README.md`, each covering
  that subproject's setup/scripts/layout/testing conventions in more
  depth than the root README. Moved `Dockerfile` and `docker-compose.yml`
  from the repo root into `deploy/` (alongside the existing systemd
  unit) — the Docker build context is unchanged (still the repo root;
  `deploy/docker-compose.yml` now sets `build.context: ..` and
  `dockerfile: deploy/Dockerfile`, and every documented command uses
  `-f deploy/... --project-directory .`). Moved `CHANGELOG.md` (this
  file) from the repo root into `docs/`, alongside the rest of the
  documentation set. `docs/DEPLOYMENT.md` and the root `README.md`
  updated for the new paths/commands.
