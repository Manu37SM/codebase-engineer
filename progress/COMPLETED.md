# Completed

## Phase 0 — Product Architecture (2026-08-18)
- Authored `/docs/PRD.md`, `/docs/ARCHITECTURE.md`, `/docs/FEATURE.md`,
  `/docs/ROADMAP.md`, `/docs/SECURITY.md`, `/docs/TESTING.md`, `/docs/ADR.md`,
  `/docs/AI_MODE.md`, `/docs/MONETIZATION.md`.
- Authored `/progress/CURRENT.md`, `/progress/COMPLETED.md`,
  `/progress/BLOCKED.md`, and root `README.md`.

## Phase 1 — Project Scaffold (2026-08-18)
- Created repository structure (`backend/`, `frontend/`, `docs/`, `progress/`).
- Backend: Fastify + TypeScript scaffold with a `/api/v1/health` endpoint,
  SQLite connection module, minimal migration runner, and an initial migration
  creating `schema_migrations` and placeholder core tables.
- Frontend: React + TypeScript + Vite + Tailwind scaffold with a navigation
  shell for the nine top-level sections (Dashboard, Repositories,
  Architecture, Findings, Changes, Tests, Audit, AI Mode, Settings).
- Smoke tests added and passing (Vitest, both backend and frontend).

## Phase 2 — Repository Discovery (2026-08-18)
- Path sandboxing utility (`backend/src/security/paths.ts`): resolves and
  rejects any path that would escape a project root; rejects non-existent or
  non-directory roots. 9 tests, including textual-prefix and absolute-path
  traversal attempts.
- Gitignore-aware file walker (`backend/src/discovery/fileWalker.ts`):
  default excludes (`.git`, `node_modules`, `target`, `build`, `dist`, etc.)
  plus root-level `.gitignore`, using the `ignore` package. Does not follow
  symlinks. Skips binary/oversized files when reading content.
- Detectors: languages (Java/JavaScript/TypeScript with approximate LOC),
  build system (Maven, npm, Gradle-detected-only), package manager (npm,
  pnpm, yarn via lockfile), frameworks (evidence-based — React, Vue, Angular,
  Next.js, Vite, Express, Fastify, NestJS, Svelte, Spring Boot, Spring,
  Quarkus, Micronaut), Git (repository presence, branch, working-tree status
  via `git` CLI through `execFileSync`, no shell interpolation).
- Orchestrator (`backend/src/discovery/index.ts`) composes all detectors into
  a `DiscoveryResult`.
- Persistence: `project` and `repository_snapshot` repository functions
  (`backend/src/db/projectRepo.ts`).
- API: `POST /api/v1/projects`, `GET /api/v1/projects`,
  `GET /api/v1/projects/:id`, `POST /api/v1/projects/:id/discover` — root
  path validated against traversal/non-existence before registration;
  duplicate root-path registration rejected (409).
- Tests: 21 backend tests total (was 2) across 5 suites — fixture-repo-based
  discovery tests (JS/TS/npm project, Maven/Java project, Git status
  transitions), path-sandboxing tests, and full API integration tests
  (register → discover → fetch, 404/400/409 cases). All passing. Verified
  manually end-to-end against a live server + curl in addition to the
  automated suite.

## Phase 3 — Repository Indexing (2026-08-18)
- Extended the file walker to merge nested `.gitignore` files (previously
  root-only), rewriting each nested file's patterns relative to its own
  directory before adding them to a single shared matcher. Documented as an
  approximation of real git semantics, not a byte-identical reimplementation.
- Indexer (`backend/src/indexer/`): per-file language, approximate LOC, size,
  sha256 content hash, test-file classification (path/filename heuristics —
  `*.test.*`, `*.spec.*`, `/test(s)/`, Java `*Test.java`/`*Tests.java`/
  `*IT.java`), generated-file classification (filename pattern or in-content
  marker like `@generated`/`DO NOT EDIT`), and regex-based import extraction
  for JavaScript/TypeScript (`import`/`export … from`/`require(...)`) and
  Java (`import`/`import static`).
- DB: migration `002_file_imports.sql` adds an `imports` column to `file`;
  `backend/src/db/fileRepo.ts` does a transactional full replace of a
  project's file rows on each reindex (no accumulation of stale rows for
  deleted/moved files).
- API: `POST /api/v1/projects/:id/index`,
  `GET /api/v1/projects/:id/files` (language/isTest filter, limit/offset).
- Tests: 30 backend tests total (was 21) — added indexer tests (language/
  LOC/hash/classification, hash changes on content change, Java import
  extraction, Java test-file naming), nested-`.gitignore` tests (subdirectory
  pattern, anchored-to-its-own-directory pattern, root pattern still
  applies), and API tests (index → list → reindex-after-deletion reflects
  removal, language filter). All passing. Verified manually end-to-end
  against a live server: registered a project, ran `/index`, confirmed
  `/files` response shape and import extraction, confirmed
  `/health.migrationsApplied` reached 2.
- Documented gap (not implemented, not claimed as done): symbol-level
  extraction (classes/functions/exports) requires a real parser
  (Tree-sitter) and is deferred — see `docs/FEATURE.md`.

## Phase 4 — Dashboard (2026-08-18)
- Frontend API client (`frontend/src/lib/api.ts`) — typed wrappers over every
  Phase 2/3 endpoint, throws a typed `ApiError` carrying the server's message
  and HTTP status.
- `ProjectContext` (`frontend/src/context/ProjectContext.tsx`) — loads the
  project list on mount, tracks the selected project, persists the selection
  in `localStorage` so it survives a page reload.
- Repositories page: register a project by absolute path (client + server
  validation surfaced in the UI), list registered projects, select one, and
  trigger a "Scan" (discover + index in sequence) with a result summary.
  Built alongside Phase 4 — see scope note in `progress/CURRENT.md`.
- Dashboard page: real repository summary wired to
  `GET /api/v1/projects/:id` + `GET /api/v1/projects/:id/files` — repo name/
  path, per-language file count + approximate LOC, build system, package
  managers, frameworks, Git branch, working-tree clean/dirty, total file
  count, test file count, last-scan timestamp. Distinct states for "no
  project selected" and "selected but never scanned" rather than a blank or
  misleading dashboard.
- Tests: 12 frontend tests (was 1) — API client (request shape, query-string
  building, error surfacing), Repositories page (empty state, register flow,
  validation error, scan flow), Dashboard page (all three states: none
  selected / not yet scanned / full summary), and the nav-shell smoke test
  updated to account for the app now doing a real (mocked-in-test) data
  fetch on mount. All passing, confirmed stable across repeated runs. Full
  frontend + backend rebuild and full test suites (12 + 30 = 42 tests) both
  green. Manually verified against a live backend with curl (project
  registration round-trip).

## Phase 5 — Architecture Explorer (2026-08-18)
- Import resolution (`backend/src/architecture/resolveImports.ts`): resolves
  relative JS/TS imports (including NodeNext-style `.js` specifiers pointing
  at `.ts` source — see bug note below) to actual indexed files, and Java
  fully-qualified imports to files by package+classname suffix match.
  Wildcard Java imports and bare/npm specifiers are reported as "external",
  never fabricated as internal edges.
- Aggregation (`backend/src/architecture/aggregate.ts`): groups files into
  "modules" by the first N path segments (configurable depth, 1-3 in the
  UI), rolls up file-level import edges into weighted module-level edges,
  drops intra-module edges (not architecturally meaningful), summarizes the
  top 30 external dependencies by reference count.
- API: `GET /api/v1/projects/:id/architecture?depth=N`, reading from
  already-indexed `file` rows (`listAllProjectFiles`, unpaginated by design
  — the explorer needs the full graph).
- Frontend: Architecture page — depth selector (1/2/3), module table
  (files/tests/LOC/depends-on/used-by), external dependencies panel.
  Deliberately a table, not a rendered node/edge diagram — see
  `docs/FEATURE.md` "Visual dependency graph" row and the scope note in
  `progress/CURRENT.md`.
- Bug caught by dogfooding, not just fixtures: initial import resolution
  returned zero edges when run against this codebase's own `backend/src`,
  because every real import here is written `"./foo.js"` (NodeNext/ESM
  TypeScript convention) while the actual file on disk is `foo.ts`. Fixed by
  stripping a trailing JS extension before resolution; added a regression
  test and a second real-repo verification pass confirming ~18 real edges
  now resolve correctly.
- Also fixed during dogfooding: module `totalLoc` was initially summing LOC
  across all files including non-source files like `package-lock.json`,
  making a module's line count wildly misleading (2,936 "LOC" from a
  lockfile). Restricted to recognized-language files, consistent with the
  Dashboard's LOC stat.
- Tests: 44 backend tests (was 30) — import resolution (JS relative +
  extensionless + directory-index + NodeNext `.js`→`.ts`, Java FQN + package
  mismatch + wildcard exclusion), aggregation (depth collapsing, intra-module
  edge exclusion, external dependency ranking, root-file `(root)` module),
  and API tests (empty-before-index, depth param, invalid depth). 16
  frontend tests (was 12) — Architecture page's three states plus the depth
  control triggering a refetch. Full frontend + backend rebuilds and full
  suites green (44 + 16 = 60 tests), confirmed stable across repeated runs.

## Phase 6/7 — Deterministic Analysis Engine + Findings System (2026-08-18)
- Analysis engine (`backend/src/analysis/`): `buildAnalysisContext` walks the
  repo fresh off disk (raw text needed by rules, not just the Phase 3 index),
  populating language/LOC/test-classification/generated-classification/
  imports per file. Five rules, each producing `Finding` records with
  rule_id/severity/category/file/line/evidence/explanation/recommendation:
  - `large-file` — ≥400 LOC (medium) / ≥800 LOC (high), skips generated files.
  - `large-function` — brace-balance heuristic over function/arrow
    signatures, ≥60 lines (high if >120), skips past a matched function's
    span so nested functions aren't double-reported.
  - `todo-fixme-density` — ≥3 TODO/FIXME/XXX markers in one file, evidence
    cites actual line numbers.
  - `missing-test-file` — usage-based (BFS transitive closure over the
    import graph starting from test files, reusing `resolveImports` from the
    Architecture explorer) OR naming-convention fallback
    (`<name>.test.*`/`<name>.spec.*`, Java `<Name>Test.java`). See dogfooding
    bug note below — this is the rule that needed two rounds of fixing.
  - `hardcoded-secret` — private-key blocks and AWS access key IDs
    (critical), generic `api_key=`/`secret=`/`password=`/`token=` assignments
    (high); evidence always redacted (first 4 + last 2 chars); skips test
    files.
- Persistence (`backend/src/db/findingRepo.ts`): `replaceProjectFindings`
  (transactional delete+insert scoped to `source = 'deterministic'`),
  `listFindings` (severity/category filters, severity-ranked ordering),
  `createAnalysisRun`/`finishAnalysisRun`/`getLatestAnalysisRun`.
- API: `POST /api/v1/projects/:id/analysis` (runs the engine, replaces
  findings, tracks the run), `GET /api/v1/projects/:id/findings` (severity/
  category/limit/offset filters, includes the latest run).
- Frontend: Findings page — severity-colored badges, severity/category
  filter dropdowns, a "Run Analysis" button, and distinct states for no
  project selected / analysis never run / no findings match filters / a
  populated list (file:line, evidence, explanation, recommendation per
  finding). Wired into the `/findings` route, replacing the earlier
  placeholder.
- Dogfooding bug (most significant of this phase): the initial
  `missing-test-file` rule used naming-convention matching only, producing
  16-18 false positives against this project's own `backend/src` because
  its tests are organized by topic (`discovery.test.ts` covers `git.ts`,
  `languages.ts`, `buildSystem.ts`, `frameworks.ts` together), not 1:1
  file-to-test naming. A first fix (checking direct test-file imports via
  the import graph) reduced this to 16 but still missed files only
  reachable transitively through an orchestrator module a test imports
  directly. Fixed with a full BFS transitive closure over the import-graph
  adjacency list starting from every test file. Reverified against
  `backend/src`: 18 → 2 findings, both genuine, 0 false positives. Two
  regression tests added.
- Tests: 63 backend tests (was 44) — 17 new analysis-engine tests (per-rule
  true/false positives, plus the two missing-test-file regression cases)
  and analysis/findings API round-trip + 404 tests. 22 frontend tests (was
  16) — 6 new Findings page tests (no project selected, never run, filtered
  empty, populated list with severity badge, run-analysis button flow,
  severity filter dropdown). An early version of the new frontend test
  suite was intermittently flaky (`findByText` resolving to an element
  right before a subsequent re-render detached it); fixed by switching
  those assertions to `waitFor(() => expect(screen.getByText(...))
  .toBeInTheDocument())`, confirmed stable across 5+ consecutive full-suite
  runs. Full frontend + backend rebuilds and full suites green
  (63 + 22 = 85 tests).

## Phase 8 — Git Analysis (2026-08-18)
- Git analysis service (`backend/src/git/`): `getRecentCommits` (commit log
  via `git log` with ASCII unit/record-separator parsing — not a printable
  delimiter like "|", which a commit subject could contain and corrupt),
  `getFileChurn` (commit count per file within a configurable day window via
  `git log --since --name-only`), `getUncommittedChanges` (staged+unstaged
  diff stat vs HEAD via `git diff HEAD --numstat`, returns `null` rather
  than a fabricated empty summary when there's no HEAD yet). `analyzeGit`
  orchestrator composes these with the existing Phase 2 `detectGit`
  (branch/working-tree status). All git invocations via `execFileSync`, no
  shell interpolation. Computed live per request, not persisted — Git state
  changes on every commit, so (like the Phase 5 Architecture explorer) a
  cached view would mislead; there's also no `git_analysis` table in the DB
  model.
- API: `GET /api/v1/projects/:id/git` (optional `commitLimit`/`churnDays`
  query params, validated, 400 on invalid values), returns
  `isGitRepository: false` with empty arrays (not an error) for a project
  whose root has no `.git` directory.
- Frontend: a new "Git activity" section on the Dashboard page — recent
  commits, most-churned files, uncommitted-changes summary. Fetched
  independently of the main dashboard summary so a Git failure (e.g. `git`
  not on PATH) can't block the rest of the dashboard from rendering.
  Deliberately not a new top-level nav item — Git Analysis has its own PRD
  bullet and API endpoint but was never one of the product's 9 designed nav
  sections, so it was folded into the existing Dashboard page instead of
  inventing UI structure not in the design. Documented as a scope decision
  in `progress/CURRENT.md`.
- Tests: 77 backend tests (was 63) — 11 new git-analysis tests against real
  temp Git repositories (`git init` + real commits via `execFileSync`, not
  mocked), covering commit parsing/ordering/limit, a commit subject
  containing "|" to prove the delimiter choice matters, churn counting and
  window exclusion (using `GIT_AUTHOR_DATE`/`GIT_COMMITTER_DATE` to backdate
  a commit reliably rather than relying on tight real-time timing), diff
  stats for staged+unstaged+new files, and the non-git-repo case. Plus 3 new
  API tests (git round-trip, non-git project, invalid `commitLimit` + 404).
  25 frontend tests (was 22) — 3 new Dashboard tests (non-git message,
  populated Git activity section, Git failure not blocking the rest of the
  page). Also manually verified end-to-end against a live server + curl
  using a freshly created temp git repo, since this project's own sandboxed
  checkout isn't itself under Git (no `backend/src` self-dogfood was
  possible for this phase specifically). Full frontend + backend rebuilds
  and full suites green (77 + 25 = 102 tests), frontend suite confirmed
  stable across 5+ consecutive runs.

## Phase 9 — Test Runner (2026-08-18)
- Command detection (`backend/src/testrunner/detect.ts`): Maven (`mvn -B -q
  test`) when `pom.xml` exists; npm/pnpm/yarn (whichever lockfile is
  present, default npm) `test` script when `package.json` has one, labeled
  framework `vitest` when that's a declared dependency, `npm-script`
  otherwise. Explicitly reports `supported: false` with a reason — never a
  fabricated command — for: no test script, the default npm-init
  placeholder test script, Gradle-only projects (matches the existing
  "Gradle-detected-only" scope note from Phase 2), and repos with no
  recognized build system.
- Execution (`backend/src/testrunner/run.ts`): `child_process.spawn` with
  an argv array (never a shell string — nothing in a project's own scripts
  can inject an additional command), spawned in its own process group
  (`detached: true`) so a timeout can kill the whole tree via
  `process.kill(-pid)`, not just the immediate `npm`/`mvn` process. 5MB
  captured-output cap, 5-minute default timeout (configurable). Always
  resolves with a result object — a failing test suite is expected output,
  not an exception.
- Output parsing (`backend/src/testrunner/parse.ts`): Vitest's "Tests ...
  passed/failed/skipped" summary line and Maven Surefire's "Tests run: N,
  Failures: N, Errors: N, Skipped: N" line(s), summed across modules for
  multi-module Maven builds. Unrecognized frameworks get `null` counts
  (honestly unknown), never zeros or guesses.
- Persistence: extended the `test_run` table (created empty in Phase 0,
  first real writer here) with `status`/`reason` columns via a new
  migration (003 — 001 was already applied on the user's local DB, so
  extended rather than edited in place). `stdout_ref`/`stderr_ref` hold raw
  output directly, documented as such (no blob-storage subsystem exists).
- API: `POST /projects/:id/tests/run` (synchronous, bounded by the runner's
  timeout — same pattern as Phase 6/7's analysis-run endpoint, not a
  background job queue), `GET /projects/:id/tests` (history, output
  omitted from the list payload to stay light), `GET /projects/:id/tests/:runId`
  (single run, full output included).
- Frontend: Tests page — run button, latest-run status badge/summary
  (pass/fail/skip counts, duration) or unsupported reason, expandable
  stdout+stderr, clickable run history.
- Two real dogfooding bugs, both caught by running the actual `npm test`
  command against this project's own `backend/` and `frontend/` (not just
  hand-written fixture strings):
  1. The Vitest parser's regex was anchored on a line starting with
     "Tests", but Vitest's default reporter colors that label, wrapping it
     in an ANSI escape code — the line never matched, so counts silently
     came back null even though the tests genuinely ran and passed. Fixed
     by stripping ANSI escape codes before parsing; reverified against this
     project's own backend (77/77) and frontend (25/25) suites, both now
     parsed correctly.
  2. The first execution approach used `execFile`'s built-in `timeout`
     option, which only signals the immediate child process. Since `npm
     test` spawns a grandchild (the real test binary), a timeout would
     leave that grandchild running, orphaned. Fixed by switching to
     `spawn` with `detached: true` plus a manual timer that kills the whole
     process group; covered by a test that starts a genuinely long-running
     command and asserts it's actually stopped.
- Tests: 100 backend tests (was 77) — 20 new (`detectTestCommand` per build
  system/edge case, both parsers including the ANSI-colored regression
  case, and `runTests` executing real subprocesses: a real passing script,
  a real failing script with a real non-zero exit code, and a real
  long-running script that gets killed on timeout — none of these three
  mocked) plus 4 new API tests (run → persist → fetch → history,
  unsupported project, 404s). 31 frontend tests (was 25) — 6 new Tests page
  tests. Full frontend + backend rebuilds and full suites green
  (100 + 31 = 131 tests), frontend suite confirmed stable across 5+
  consecutive runs.

## Phase 10 — Dependency & Security Analysis (2026-08-18)
- Dependency analysis (`backend/src/dependencies/`): `analyzeDependencies(root)`
  is computed live off the manifest/lockfile each call, same pattern as
  Architecture (Phase 5) and Git analysis (Phase 8) — not persisted. Maven
  branch (pom.xml present): regex extraction of `<dependency>` blocks from
  `pom.xml` (`groupId:artifactId` naming, unresolved `${property}` version
  placeholders reported as literal text rather than resolved); duplicate-version
  detection is explicitly not attempted for Maven (would require resolving the
  full dependency tree via `mvn dependency:tree`, which this product doesn't
  invoke) — an honest note is returned instead of fabricated results. npm
  branch (package.json present): direct deps read from `dependencies` +
  `devDependencies`; duplicate-version detection parses `package-lock.json`'s
  v2/v3 flat `packages` map (keyed by `node_modules/...` paths, handling
  nested and scoped package names) and groups by package name. v1-format
  (nested-tree) lockfiles and missing lockfiles get an honest
  "not supported"/"no lockfile" note, never fabricated duplicates. `GET
  /projects/:id/dependencies` API route. Dashboard "Dependencies" section:
  direct-dependency count and a duplicate-versions list (capped display of
  10 with a "+N more" note), or the honest note when duplicates aren't
  available.
- Security analysis (`backend/src/security/scan.ts`): `scanSecurity(root)`
  runs only the security-category subset of the analysis engine
  (`SECURITY_RULES`, exported from `analysis/index.ts`) fresh off disk each
  call — a live/current view, deliberately distinct from the historical
  `GET /findings?category=security` (which only reflects the last persisted
  analysis run). `GET /projects/:id/security` API route. Three new rules
  alongside the existing `hardcoded-secret`: `env-file-committed` (flags real
  `.env`-style files, excluding `.env.example`/`.sample`/`.template`/
  `.defaults`/`.dist` templates; never echoes file contents into evidence —
  the file's presence is the finding), `permissive-cors` (flags
  `Access-Control-Allow-Origin: *` headers and `cors({ origin: true | '*' })`
  -shaped middleware config), `disabled-tls-verification` (flags
  `rejectUnauthorized: false`, `NODE_TLS_REJECT_UNAUTHORIZED=0`, Python
  `verify=False`, and an empty Java `checkServerTrusted()`).
- Real dogfooding bug, caught twice: running the 3 new rules against this
  project's own real `backend/`/`frontend/` source produced 4 false
  positives, all pointing at the rules' own definition files — the label
  strings and an explanatory comment literally spelled out the exact
  vulnerable syntax (`"rejectUnauthorized: false"`,
  `"NODE_TLS_REJECT_UNAUTHORIZED=0"`, `"verify=False"`, `origin: true}`),
  which matched the rules' own detection regexes. Fixed (round 1) by
  paraphrasing all labels/comments to describe the pattern rather than
  literally quote it. Rereunning the dogfood check surfaced a second-order
  case: the comment added to explain the round-1 fix itself quoted the
  exact offending string it was describing, so it *also* matched. Fixed
  (round 2) by rewriting that comment to describe the issue abstractly
  without quoting the vulnerable syntax anywhere in the file. Reverified:
  zero findings against this project's own real `backend/` and `frontend/`,
  and all 4 rules confirmed to still correctly fire against synthetic
  vulnerable fixtures (TS/Python/JS/`.env`) in a scratch directory.
- Tests: 122 backend tests (was 100) — 22 new (3 security rules × 3 tests
  each, `scanSecurity` × 2, npm dependency parsing/duplicates × 4, Maven
  dependency parsing × 3, no-manifest case × 1, plus 4 new API round-trip/404
  tests for both new routes). 33 frontend tests (was 31) — 2 new Dashboard
  Dependencies-section tests. One pre-existing frontend test
  ("prompts to scan when the selected repository has no snapshot yet") began
  intermittently racing once a third independent concurrent fetch
  (dependencies, alongside the existing main-snapshot and Git fetches) was
  added to the Dashboard's effect — fixed using the same `waitFor`+`getByText`
  pattern already established elsewhere in this file for exactly this kind of
  multi-fetch race, rather than the more failure-prone bare `findByText`.
  Full backend + frontend rebuilds and full suites green (122 + 33 = 155
  tests), both suites confirmed stable across 3 consecutive runs. Also
  dogfooded end-to-end against a live running server (both routes hit via
  real HTTP against this project's own registered `backend` project) in
  addition to the automated suite.

## Phase 11 — Audit Report (2026-08-18)
- `backend/src/audit/build.ts`: `buildAuditReport(db, project)` aggregates
  this product's existing data sources into one consolidated view — it runs
  no new analysis of its own. Reuses persisted data as-is (latest
  discovery/index snapshot, latest `POST /analysis` run's finding counts by
  severity/category via a new `getFindingCounts` DB query, latest test run)
  and computes the rest fresh on every call, same as their standalone
  endpoints (live security scan, live dependency analysis, live Git
  activity) — deliberately not cached or persisted, matching the
  "computed vs. persisted" split used throughout this product. Every
  "never happened yet" case (unscanned, unanalyzed, untested) is reported
  honestly as `null`/empty rather than fabricated.
- `backend/src/audit/markdown.ts`: `buildAuditMarkdown(report)` renders the
  same aggregate as a standalone Markdown document. Evidence in the
  security-findings section is included as-is — redaction happens at the
  rule layer (Phase 6 invariant), not re-checked here.
- API: `GET /projects/:id/audit` (JSON), `GET /projects/:id/audit/export`
  (`Content-Type: text/markdown`, `Content-Disposition: attachment`, so a
  browser click downloads a real file rather than navigating to raw text).
- Frontend: Audit page (replacing the Phase 0 placeholder) — repository
  snapshot stats, static-analysis finding counts, live security findings
  (first frontend surface for the Phase 10 security-scan endpoint, which
  previously had no page of its own), dependency summary, Git activity
  summary, latest test-run summary, and a "Download report (.md)" link.
- A minor real bug caught by dogfooding the exported Markdown (not by any
  test — none of the automated tests asserted on exact blank-line spacing):
  the "Security scan" section was missing a blank line before the
  "Dependencies" heading when there were zero security findings, so the two
  headings ran together. Fixed by adding the missing separator.
- Tests: 129 backend tests (was 122) — 2 new (`buildAuditMarkdown` unit
  tests: one fully-empty report exercising every "honest empty state"
  branch, one fully-populated report including a redacted security finding)
  plus 5 new API tests (full aggregate after discover+index+analysis+a
  committed `.env` file, honest nulls for a never-scanned project, 404,
  Markdown export headers/content, export 404). 39 frontend tests (was 33)
  — 4 new Audit page tests plus 2 new `api.ts` client tests
  (`getAudit`, `getAuditExportUrl`). Full backend + frontend rebuilds and
  full suites green (129 + 39 = 168 tests), both suites confirmed stable
  across 3 consecutive runs. Also dogfooded end-to-end against a live
  server: once against this project's own real `backend/` before any scan
  had ever been run (verifying every honest-empty-state branch), and once
  after running discover+index+analysis on it for real (verifying real
  snapshot/finding-count/dependency-duplicate data flows through correctly,
  including the rendered Markdown export).

## Phase 12 — AI Provider Abstraction (2026-08-18)
- `backend/src/ai/provider/types.ts`: the `AIProvider` interface
  (`listModels`, `complete`, `estimateTokens`, `checkStatus`) plus
  `AIProviderError` (carries a classified `kind` — `auth_error` /
  `rate_limited` / `unreachable` — so call sites don't re-parse HTTP status
  codes). Business logic depends only on this interface, per
  `docs/AI_MODE.md` §2 — never a vendor SDK directly.
- `backend/src/ai/provider/adapters/openaiCompatible.ts`: one real,
  working adapter — talks real HTTP (`fetch`, with a timeout + abort) to
  any server implementing the OpenAI Chat Completions API shape (`GET
  /models`, `POST /chat/completions`). This covers OpenAI itself and the
  many local/self-hosted servers (Ollama, LM Studio, vLLM) that expose an
  OpenAI-compatible surface, so one adapter serves several real backends.
  `anthropic-compatible` and `ollama` (native API) are named in
  `docs/AI_MODE.md`'s target design but have no adapter yet —
  `backend/src/ai/provider/registry.ts#createProvider` throws an honest
  "not yet supported" error for them instead of faking a stand-in, matching
  this product's stated aversion to stubs.
- `backend/src/db/aiProviderRepo.ts`: CRUD for `provider_configuration`
  (created empty in Phase 0, first real writer here). The raw API key is
  stored server-side only; `toPublic()` strips it from every response,
  replacing it with a masked preview (`maskApiKey`: first 4 / last 2
  characters, matching the redaction style `docs/SECURITY.md` §4 already
  established for secret findings) and a `hasApiKey` boolean.
- Migration 004 adds the `api_key` column `provider_configuration` needed
  but never had (the Phase 0 schema only anticipated an indirect
  `api_key_ref` into an external secrets manager this product doesn't
  have). Verified safe against an existing pre-Phase-12 database: a DB with
  only migrations 001-003 applied and a real `provider_configuration` row
  written under the old schema was opened through the real `openDatabase()`
  — the row survived intact, the new column came back `NULL`, and
  `schema_migrations` recorded 004 as applied.
- API: `GET/POST/PATCH/DELETE /api/v1/ai/providers[/:id]`, plus `POST
  .../:id/check-status` (live reachability check) and `GET .../:id/models`
  (live model listing) — both are real network calls to the configured
  provider, gated on the API being called explicitly, never triggered by
  listing/loading providers.
- Frontend: AI Mode page (replacing the Phase 0 placeholder) — add/list/
  enable/disable/delete a provider, a "Check status" button per provider
  (fires only on click), status badges. Explicit copy on the page states
  that every Free Mode feature keeps working with no provider configured,
  and that configuring/enabling a provider triggers no AI action by itself
  — matching `docs/AI_MODE.md`'s stated Free/AI Mode boundary.
- Explicitly scaffolding, not a user-facing AI feature: nothing in this
  product calls `complete()` from a real workflow yet. That starts at
  Phase 14 (finding explanation), once Phase 13 (context selection engine)
  exists to build the bounded prompt content Phase 14 would send.
- `docs/SECURITY.md` §9 updated to honestly reflect what Phase 12 actually
  delivers of §5 (AI egress control): the API-key-never-returned and
  explicit-trigger-only pieces are real and tested; the "only send the
  Context Engine's bounded selection" provision has nothing to apply to
  yet, since no phase sends a real prompt yet.
- Tests: 161 backend tests (was 129) — 32 new: 8 adapter tests run against
  a real local HTTP server speaking the OpenAI-compatible protocol (not a
  mocked `fetch`) covering listModels/complete/checkStatus and real
  401/429/timeout/malformed-response classification; 4 registry tests; 8
  repo tests (including the masking logic and the "never in `toPublic`"
  assertion); 12 API route tests (create/list/patch/delete/check-status/
  models, validation, 404s, and confirming the raw key never appears in a
  response body). 47 frontend tests (was 39) — 8 new AI Mode page tests.
  Full backend + frontend rebuilds and full suites green (161 + 47 = 208
  tests), both suites confirmed stable across 3 consecutive runs. Also
  dogfooded end-to-end against a live server and a real local HTTP double:
  create a provider → list (confirming the API key never appears in the
  response body) → check-status (`reachable`) → list models → a real
  `complete()` call returning genuine content from the double.

## Phase 13 — AI Context Selection Engine (2026-08-18)

- `selectContextForFinding()` (`backend/src/ai/context/select.ts`) builds a
  `ContextBundle` for a `Finding` target per `docs/AI_MODE.md` §3's
  selection order: directly affected file → directly relevant methods/
  functions (approximated via line-windowing around the finding, largest
  fitting window of 200/80/30/10 lines — no AST parser exists, so true
  function-boundary extraction isn't attempted rather than faked) →
  imported symbols (reusing the same `resolveImports()` graph as
  Architecture/`missing-test-file`) → known callers → associated test
  file(s) (prefers an importing test file over a naming-convention guess)
  → relevant config (the same root manifest Phase 10's dependency
  analyzer reads) → relevant Git diff hunk (new `getUncommittedDiffForFile()`,
  `backend/src/git/fileDiff.ts`, argv-safe, only offered when the file has
  real uncommitted changes). Every candidate is redacted via a new shared
  module before being counted toward the budget or returned.
- New shared modules (both zero-regression refactors of existing logic):
  `backend/src/security/secretPatterns.ts` (secret-detection patterns +
  redaction, factored out of the `hardcoded-secret` rule so AI context
  gets the exact same protection); `backend/src/ai/tokenEstimate.ts`
  (the chars/4 estimate, factored out of the Phase 12 adapter so it can't
  drift from the context selector's copy).
- API: `GET /api/v1/projects/:id/findings/:findingId/context?budgetTokens=N`
  (default 4000). Frontend: "Preview AI context" toggle per finding on the
  Findings page — a preview only, since nothing downstream consumes the
  bundle yet (Phase 14 will be the first).
- Deliberately scoped to `Finding` targets only; `TestRun`/refactor-request
  targets from `docs/AI_MODE.md` §3 are explicitly not implemented — no
  real downstream consumer exists yet.
- Genuine bug caught via test-writing: a test-file caller was being added
  to the candidate list twice (generic "known caller" loop + test-file-
  specific loop), so `Array.find()` returned the less specific reason
  first. Fixed by excluding test-file callers from the generic loop.
- Dogfooded against this project's own real `backend/`: at budget=4000,
  selected the primary file + 2 genuinely-imported files (3999/4000
  tokens), honestly excluded 5 other imports + 1 known caller + 1 test
  caller with real token-cost reasons; at budget=100, correctly excluded
  even the smallest line-window of the primary file.
- Tests: 188 backend (was 161) — 27 new: 6 `secretPatterns.test.ts`, 6
  `gitFileDiff.test.ts`, 12 `aiContextSelect.test.ts`, 3 new route tests.
  48 frontend (was 47) — 1 new (preview toggle, open + close). Full
  backend + frontend rebuilds green, both suites stable across repeated
  runs, `tsc --noEmit` and `vite build` clean.

## Phase 14 — AI Finding Explanation (2026-08-18)

- `explainFinding()` (`backend/src/ai/workflows/explainFinding.ts`) — the
  first phase to actually call `AIProvider.complete()` from a real
  user-facing workflow, using Phase 12's provider and Phase 13's context
  engine together for the first time. Builds a Phase 13 `ContextBundle`
  with the new `includeContent: true` option (Phase 13's preview API
  still defaults to the lightweight summary), assembles a system+user
  prompt from the finding summary and the bundle's real, already-redacted
  file content, calls the provider, and persists an accounting record —
  succeed or fail — via new `backend/src/db/aiRequestRepo.ts`. Strictly
  read-only: never writes to the finding or applies anything.
- Migration 005 (`005_ai_explanation.sql`) extends the Phase 0
  `ai_request`/`ai_response` scaffold tables with `finding_id` and
  `content`, so a call is traceable to what it explained and what the
  provider said. Verified against a simulated pre-existing database
  (migrations 001-004 only, real accounting rows present) — upgrades
  cleanly, no data loss.
- API: `POST /api/v1/projects/:id/findings/:findingId/explain` (the only
  route so far that spends real tokens against a real provider — explicit
  call only, never automatic; 400 if no provider is enabled; provider
  failures classified to 401/429/502) and
  `GET /api/v1/projects/:id/findings/:findingId/explanation` (read-only,
  no provider call, serves the most recent successful explanation).
- Frontend: "AI explanation" toggle per finding on the Findings page —
  shows a stored explanation or a "Generate explanation" button, visibly
  disabled (not hidden) with an inline message when no AI provider is
  enabled, matching the AI Mode page's existing Free/AI Mode boundary
  pattern.
- Dogfooded end-to-end against this project's own real `backend/` source
  and a real local HTTP double speaking the OpenAI-compatible protocol:
  full discover+index+analysis, a real provider configured and enabled,
  `/explain` called on a genuine `large-function` finding — confirmed the
  real prompt sent, the real (redacted) file content in it, the persisted
  response, and `GET .../explanation` correctly serving the same stored
  text back without a second provider call.
- Tests: 198 backend (was 188) — 10 new: 4 `aiRequestRepo.test.ts`, 2
  `explainFinding.test.ts` (including a real 401 failure path asserting
  the accounting row is marked failed and nothing is stored as a
  successful explanation), 4 new route tests. 51 frontend (was 48) — 3
  new (disabled-with-message when no provider, generate + display, and
  reusing a stored explanation without a second call). Full backend +
  frontend rebuilds green, both suites stable across repeated runs,
  `tsc --noEmit` and `vite build` clean.

## Phase 15 — AI Root-Cause Analysis (2026-08-18)

- `analyzeRootCause()` (`backend/src/ai/workflows/rootCauseAnalysis.ts`),
  sharing context-selection/provider-call/accounting plumbing with Phase
  14's `explainFinding()` via a new `runFindingWorkflow()` helper —
  factored out once a second Finding-target workflow needed identical
  bookkeeping (zero-regression refactor, full suite reverified green).
- Asks the provider for a structured `EVIDENCE:`/`INFERENCE:`/`CONFIDENCE:`
  response (distinct from Phase 14's plain prose), parsed by
  `parseRootCauseSections()`: any section not clearly present is `null`
  rather than guessed, with the full raw response always preserved. A
  real bug (the EVIDENCE regex not stopping before an absent INFERENCE:
  header, swallowing the CONFIDENCE: line into evidence bullets) was
  caught by test-writing and fixed.
- No new migration — reuses Phase 14's `ai_request`/`ai_response`
  accounting tables, distinguished by `operation_type`; verified the two
  operation types stay independent for the same finding.
- API: `POST /api/v1/projects/:id/findings/:findingId/root-cause` and
  `GET .../root-cause` (re-parses stored raw response on every fetch
  rather than persisting the parsed shape), via a new shared
  `resolveEnabledProvider()` route helper mirroring `/explain`'s error
  handling.
- Frontend: "Root-cause analysis" toggle on the Findings page — evidence
  as a bulleted list, inference as prose, kept visually distinct, with a
  collapsible raw-response fallback when parsing didn't fully succeed;
  same visibly-disabled-when-no-provider pattern as Phase 14.
- Dogfooded end-to-end against this project's own real `backend/` source
  and a real local HTTP double: real structured prompt/response on a
  genuine `large-function` finding, `GET .../root-cause` correctly served
  the same result back, and a subsequent `/explain` call on the same
  finding left the stored root-cause analysis undisturbed (and vice
  versa).
- Tests: 207 backend (was 198) — 9 new: 6 `rootCauseAnalysis.test.ts`
  (including the real EVIDENCE-parsing bug), 3 new route tests. 54
  frontend (was 51) — 3 new. Full backend + frontend rebuilds green, both
  suites stable across repeated runs, `tsc --noEmit` and `vite build`
  clean.

## Phase 16 — AI Fix Planning (2026-08-18)

- `planFix()` (`backend/src/ai/workflows/fixPlan.ts`) produces the exact
  seven-section plan docs/AI_MODE.md §5 requires (Problem, Root cause,
  Files affected, Proposed changes, Risks, Required tests, Validation
  strategy). First workflow to actively consume a previous workflow's
  output: when a successful Phase 15 root-cause analysis already exists
  for the finding, its raw text is folded into the prompt as grounding
  (`usedPriorRootCauseAnalysis` reports this); otherwise plans from the
  finding and context bundle alone.
- Generalized the structured-response parser: `parseStructuredSections()`
  (`backend/src/ai/workflows/parseStructuredResponse.ts`) replaces Phase
  15's bespoke 3-header regex pair and handles Phase 16's 7-header format
  too — each section now terminates at *any* later header (not just the
  next one), a strict superset of the fix already shipped for Phase 15's
  own EVIDENCE/CONFIDENCE bug. `rootCauseAnalysis.ts` refactored onto it,
  full suite reverified green first.
- Strictly advisory, same as Phases 14-15: a fix plan is prose, never a
  diff, and changes nothing on disk. Patch generation (Phase 17) remains
  a separate, later, human-approval-gated workflow.
- API: `POST /api/v1/projects/:id/findings/:findingId/fix-plan`,
  `GET .../fix-plan`, mirroring `/explain`/`/root-cause`'s shape via the
  shared `resolveEnabledProvider()` route helper.
- Frontend: "Fix plan" toggle on the Findings page — all seven sections
  plus files-affected as a bulleted list, raw-response fallback, same
  visibly-disabled-when-no-provider pattern.
- Dogfooded end-to-end against this project's own real `backend/` source
  and a real local HTTP double: real root-cause analysis followed by a
  real fix-plan call on the same `large-function` finding — confirmed the
  fix-plan prompt contained the stored root-cause text, all seven
  sections parsed correctly, and `GET .../fix-plan` served the identical
  result back.
- Tests: 224 backend (was 207) — 17 new: 9 `parseStructuredResponse.test.ts`,
  5 `fixPlan.test.ts`, 3 new route tests. 57 frontend (was 54) — 3 new.
  Full backend + frontend rebuilds green, both suites stable across
  repeated runs, `tsc --noEmit` and `vite build` clean.

## Phase 17 — AI Patch Generation (2026-08-18)

- The first AI-Mode phase whose output could eventually change a file on
  disk, so unlike Phases 14-16 (single request/response calls), it's a
  real persisted state machine: `pending_approval -> approved -> proposed`
  (has a diff), or `pending_approval -> rejected`, backed by the `patch`/
  `patch_review` tables (Phase 0 scaffold, empty/unused until now). Both
  approval gates docs/AI_MODE.md §4 requires are enforced server-side
  against the real `patch.status` column, not just in the UI — verified
  by a test and the dogfood run calling `/generate` before `/approve` and
  confirming a 400 refusal.
- Migration 006 relaxes `patch.diff_text` from `NOT NULL` to nullable
  (table-rebuild pattern, SQLite has no `ALTER COLUMN`). Migration-safety
  verification (simulate a real pre-existing database, same methodology
  used since Phase 12) caught a real bug: `DROP TABLE patch` while
  `foreign_keys=ON` cascade-deleted real `patch_review` rows before the
  rename completed. Fixed at the shared migration-runner level
  (`backend/src/db/index.ts`'s `runMigrations()` now toggles
  `PRAGMA foreign_keys` OFF/ON around the whole batch of pending
  migrations, since the pragma can't change inside a transaction) —
  benefits every future rebuild-style migration, not just this one.
  Reverified with an expanded safety script confirming data survival and
  correct FK re-enablement.
- `generatePatch()` (`backend/src/ai/workflows/generatePatch.ts`) reuses
  `runFindingWorkflow()` unchanged from Phases 15-16 and folds the
  finding's latest successful Phase 16 fix plan into the prompt as
  grounding when one exists (`usedFixPlan`, mirroring
  `usedPriorRootCauseAnalysis`). Asks the provider for ONLY a unified
  diff, or `NO_PATCH: <reason>` if it can't produce one — preserved
  verbatim. The diff is stored raw, exactly as returned: never parsed,
  never validated (no `git apply --check`), never written to any file.
  Applying an approved diff to disk is out of scope until Phase 18.
- Redaction scope clarified (test-authoring correction, not a product
  bug): secret redaction applies to context sent TO the provider, not to
  the provider's own returned diff — a generated diff can legitimately
  quote a secret being removed, exactly as a real diff would.
- API: `POST/GET /api/v1/projects/:id/findings/:findingId/patches`
  (create/list), `GET /api/v1/projects/:id/patches/:patchId`,
  `POST .../approve` and `POST .../reject` (recording a `patch_review`
  row), `POST .../generate` (the only route that spends tokens, gated on
  `status === "approved"`).
- Frontend: "Patches" toggle on the Findings page — "Create patch
  (requires a fix plan)" button, per-patch Approve/Reject while
  `pending_approval`, "Generate diff" once `approved` (disabled with an
  explanatory message when no provider is enabled), "Rejected before
  generation." message if rejected, raw diff rendered in a `<pre>` block
  once generated.
- Dogfooded end-to-end against this project's own real `backend/`
  directory (not a synthetic fixture): registered the real project, ran a
  real discover+index+analysis (11 real findings), generated a real fix
  plan for a genuine `large-function` finding, then walked
  create -> premature-generate-refused(400) -> approve -> generate against
  a real local HTTP double — confirmed `usedFixPlan: true` and the diff
  persisted and was re-fetchable.
- Tests: 235 backend (was 224) — 11 new: 4 `patchRepo.test.ts`, 3
  `generatePatch.test.ts`, 5 new route tests (incl. the full lifecycle and
  reject-then-refuse-generate). 62 frontend (was 57) — 5 new (empty
  state + create, approve, reject, generate + diff render, disabled when
  no provider). Full backend + frontend rebuilds green, both suites
  stable across repeated runs, `tsc --noEmit` and `vite build` clean.

## Phase 18 — Diff Review & Approval Workflow (2026-08-18)

- The first phase in the entire product to actually write to a file on
  disk. Implements docs/AI_MODE.md §4's second and final human-approval
  gate ("Diff Review -> Human Approval -> Apply Patch"), extending Phase
  17's state machine: `proposed -> approved_for_apply -> applied`, or
  `proposed -> rejected` (a second, separate rejection distinct from
  Phase 17's pre-generation reject). `applied`/`failed` now match
  docs/ARCHITECTURE.md §3's originally-specified status enum in full.
- `POST .../approve-apply` / `POST .../reject-apply` mirror Phase 17's
  first-gate routes exactly, requiring `status === "proposed"` and
  recording a `patch_review` row (`approved_for_apply`/
  `rejected_after_review`). `POST .../apply` — the only route that writes
  to disk — requires `status === "approved_for_apply"` (or `"failed"`,
  allowing a mechanical retry without re-approving the underlying
  decision that the change is worth applying).
- `applyPatchToDisk()` (`backend/src/patch/applyPatch.ts`) always runs a
  real `git apply --check` dry run first and only performs the real write
  if that succeeds — the validation Phase 17 explicitly deferred, since a
  diff isn't guaranteed to still apply cleanly against a working tree
  that changed since generation. Pipes the diff via stdin to a real
  `git apply` using `execFileSync` with an argv array, same convention as
  every other Git integration in this product.
- A failed apply is a normal, informative outcome (same convention as the
  Free Mode test runner reporting a failed test run), not an HTTP error:
  200 with `status: "failed"` and the real `git apply` stderr in a new
  `apply_error` column. Migration 007 (`ALTER TABLE patch ADD COLUMN
  apply_error`) verified safe against a simulated pre-existing database
  with real Phase 17 patch/patch_review data already present.
- Frontend: the "Patches" panel gained "Approve diff for apply"/"Reject"
  buttons on a `proposed` patch, an "Apply patch" button once
  `approved_for_apply`, an "Applied to disk." success message, and a
  "Retry apply" button plus the real error text on `failed`.
- Deliberately still scoped: applying a patch never automatically re-runs
  tests, runs AI self-review, or does anything else in docs/AI_MODE.md
  §4's later workflow steps — those are Phase 19-21 territory.
- Dogfooded end-to-end against a real scratch COPY of this project's own
  `backend/src` (never the live source tree): registered the copy, ran a
  real analysis (53 findings), generated a real fix plan and a real diff
  for a genuine `large-function` finding, confirmed the second gate
  refused a premature `/apply`, then approved and applied — confirmed the
  real target file on disk was genuinely modified with the AI-generated
  content.
- Tests: 245 backend (was 235) — 10 new: 4 `applyPatch.test.ts` (real
  temp git repos, incl. a real successful write, a real dry-run-catches-
  drift failure, a malformed-diff clean failure), 6 new/extended patch
  tests (2 patchRepo, 4 route tests incl. the full second-gate lifecycle
  with a real file changing on disk). 66 frontend (was 62) — 5 new. Full
  backend + frontend rebuilds green, both suites stable across repeated
  runs, `tsc --noEmit` and `vite build` clean.

## Phase 19 — AI Test Generation (2026-08-18)

- Implements docs/AI_MODE.md §1's "AI-generated tests (reviewed &
  executed, not trusted on compile alone)" — the defining difference from
  Phase 17's patch generation: the final step doesn't just write a file,
  it immediately runs the project's real, existing test command (Phase 9)
  against it and persists a real `test_run` row.
- Mirrors Phases 17-18's two-gate `patch` state machine shape via new
  `generated_test`/`generated_test_review` tables (migration 008, purely
  additive, verified safe against a simulated pre-existing database with
  real Phase 17/18 data): `pending_approval -> approved -> proposed ->
  approved_for_write -> written | passed | failed_tests`, or
  `-> rejected` at either gate.
- Only ever creates a NEW file — `/write-and-run` refuses to overwrite
  any path that already exists (after a `resolveWithinRoot` path-
  traversal check on the AI-proposed path), so there's no dry-run-
  against-drift case the way patch apply has.
- `generateTest()` (`backend/src/ai/workflows/generateTest.ts`) reuses
  `runFindingWorkflow()` unchanged and folds the finding's latest Phase
  16 fix plan's "Required tests" section into the prompt as grounding
  when one exists (`usedFixPlan`). Asks for `TARGET_PATH:`/`TEST_CODE:`
  via the shared `parseStructuredSections()` parser, or `NO_TEST:
  <reason>` preserved verbatim. Strips a markdown code fence wrapping the
  whole TEST_CODE section.
- A real test command exit code 0 marks the generated test `passed`;
  nonzero marks `failed_tests`; no supported command detected marks
  `written` (honest degrade, not a failure). All three are valid retry
  entry points for `/write-and-run` without re-approving the code.
- API: `POST/GET /api/v1/projects/:id/findings/:findingId/generated-tests`,
  `GET .../generated-tests/:testId`, `POST .../approve`, `POST
  .../reject`, `POST .../generate`, `POST .../approve-write`,
  `POST .../reject-write`, `POST .../write-and-run`.
- Frontend: a "Generated tests" toggle on the Findings page mirroring
  "Patches"' shape — create/approve/reject, generate, a second
  approve/reject-for-write gate showing the generated code, then
  write-and-run/re-run, with distinct written/passed/failed messages.
- Dogfooded end-to-end against a real scratch COPY of this project's own
  `backend/` (node_modules symlinked so the real `vitest` binary actually
  runs, never the live source tree): registered the copy, ran a real
  analysis (9 findings), generated a real fix plan and real test code for
  a genuine `large-function` finding, confirmed the second gate refused a
  premature write, then approved and ran write-and-run — the real
  generated test file landed on disk and the real, full `vitest` suite
  (262 tests) genuinely executed and passed.
- Tests: 261 backend (was 245) — 16 new: 7 `generateTest.test.ts` (incl.
  `parseGeneratedTest` unit tests for code-fence-stripping and NO_TEST),
  5 `generatedTestRepo.test.ts`, 4 new route tests (incl. the full real
  lifecycle with a real file written and a real command executed, an
  overwrite-refusal case, and a reject-then-refuse-write case). 72
  frontend (was 66) — 6 new. Full backend + frontend rebuilds green, both
  suites stable across repeated runs, `tsc --noEmit` and `vite build`
  clean.

## Windows Cross-Platform Verification Pass (2026-08-19)
- The user paused phase progression to test the real build/test output on
  their actual Windows machine — this session's own environment is
  Linux, so this was the project's first real cross-platform
  verification, and it caught four genuine bugs no amount of Linux-only
  testing/dogfooding could have:
  1. `backend package.json`'s `copy:migrations` used POSIX-only shell
     syntax (`mkdir -p && cp *.sql`), failing under Windows' cmd.exe.
     Fixed with a plain Node.js script (`backend/scripts/copy-migrations.mjs`).
  2. `applyPatch.ts`'s real `git apply` calls silently rewrote `\n` to
     `\r\n` due to the user's global `core.autocrlf=true` git config.
     Fixed by passing `-c core.autocrlf=false -c core.safecrlf=false` as
     config overrides on both the dry-run and the real apply.
  3. `testrunner/run.ts` couldn't spawn `.cmd`-wrapped `npm`/`pnpm`/`yarn`
     on Windows (bare `node:child_process` `spawn` without `shell: true`),
     and its timeout-kill used a POSIX-only process-group kill. Fixed by
     switching to the `cross-spawn` package and adding a
     `process.platform === "win32"` branch using `taskkill /pid <pid> /t
     /f` for the tree kill.
  4. Frontend `Dashboard.test.tsx` had a real timing race (synchronous
     assertions after a single `findByText`, racing a multi-step async
     page load) — fixed by wrapping the assertion group in `waitFor`.
- All four verified via full rebuild + full test suite (backend 261/261,
  frontend 72/72) in this session's Linux environment, stable across
  repeated runs; the CRLF and `taskkill` code paths themselves can't be
  executed outside real Windows, so final confirmation was left to the
  user's own `npm test` run there.

## Phase 20 — AI Failure Diagnosis (2026-08-19)
- Implements docs/AI_MODE.md §4's "(if failure) AI Diagnosis" workflow
  step — the first AI-Mode feature targeting a failed `TestRun` instead
  of a `Finding` (the `TestFailureTarget` kind docs/AI_MODE.md §3 named
  but deferred since Phase 13). Read-only: never writes to the test run
  or proposes an applied change; an actual fix still goes through the
  existing fix-plan → patch-generation → diff-review flow.
- New `selectContextForTestFailure()` builds a context bundle grounded
  entirely in the failed run's own real captured stdout/stderr: the
  output excerpt is always the primary item (truncated-to-fit rather than
  ever excluded), then real indexed files whose path text is actually
  found inside that output (test-files-first, then by mention count —
  never guessed from the command alone), then the top-matched failing
  test file's own imports.
- `diagnoseFailure()` asks the provider for
  `LIKELY_CAUSE:`/`EVIDENCE:`/`SUGGESTED_DIRECTION:` via the shared
  `parseStructuredSections()` parser — deliberately no diff or code in
  the response, just a hypothesis, its evidence, and a fix direction.
- Migration 009 adds `ai_request.test_run_id` (nullable, purely
  additive — anticipated by migration 005's original comment), verified
  safe against a simulated pre-existing database with real accounting
  data already present.
- API: `POST /api/v1/projects/:id/tests/:runId/diagnose` (only callable
  when the run's status is `failed`), `GET .../diagnosis` (read-only,
  reparses the stored raw response on every fetch, never calls a
  provider).
- Tests page: a "Diagnose failure" action appears only on a `failed` run,
  mirroring the Findings page's root-cause-analysis panel shape.
- Dogfooded end-to-end against a real scratch COPY of this project's own
  `backend/` (node_modules symlinked, never the live source tree):
  deliberately broke a real function (`estimateTokens()`'s divisor),
  ran the real, full `vitest` suite via the real test runner (278 tests,
  2 real failures), diagnosed the real failure against a real local HTTP
  AI-provider double, and confirmed the context bundle actually selected
  the real captured output plus real files whose paths appeared in it.
- Tests: 278 backend (was 261) — 20 new: 7
  `selectContextForTestFailure.test.ts`, 6 `diagnoseFailure.test.ts`
  (incl. a real-HTTP-server provider-failure case that still persists an
  honest failed accounting row), 5 new route tests (incl. a real failed
  `npm test` run genuinely executed and diagnosed end to end), 1 new
  `aiRequestRepo.test.ts` case proving `test_run_id`- and
  `finding_id`-keyed accounting stay independent. 76 frontend (was 72) —
  4 new. Full backend + frontend rebuilds green, both suites stable
  across repeated runs, `tsc --noEmit`/`tsc -b` and `vite build` clean.

## Phase 21 — AI Self-Review (2026-08-19)
- Implements docs/AI_MODE.md §6's self-review checklist — correctness,
  scope creep, regressions, security, missing tests, unnecessary
  complexity, consistency with existing architecture — run against a
  real proposed patch's real diff. Purely advisory per §6's own wording:
  never changes the patch's `status`, never a precondition for
  `/approve-apply` or `/apply`; can be requested at any point once the
  patch has a real diff, including more than once for the same diff.
- `selfReviewPatch()` reuses `runFindingWorkflow` (self-review's
  grounding is the same Finding + context-bundle shape patch generation
  already uses) — `runFindingWorkflow` gained an optional `patchId`
  passthrough so the accounting row is scoped to the exact diff
  reviewed, not just "the finding". Asks for all seven checks via the
  shared `parseStructuredSections()` parser, each expected as `STATUS:
  note`; an off-format check keeps `status: null` but preserves the
  text as `note` rather than dropping it.
- Migration 010 adds `ai_request.patch_id` (nullable, purely additive —
  a finding can have several patches and a patch can be regenerated more
  than once, so `finding_id` alone would blur reviews of different
  diffs together), verified safe against a simulated pre-existing
  database with real accounting data already present.
- API: `POST /api/v1/projects/:id/patches/:patchId/self-review` (400 if
  no diff yet), `GET .../self-review` (read-only, reparses on every
  fetch, never calls a provider).
- Findings page: an "AI self-review" toggle inside each patch's card
  (visible only once it has a diff) — a colored pass/concern/fail chip
  and note per check, raw response in a collapsible details block.
- Dogfooded end-to-end against a real scratch COPY of this project's own
  `backend/` (a genuinely introduced hardcoded-secret file, never the
  live source tree): a real finding, a real fix plan, a real generated
  diff against a real local AI-provider double, self-review correctly
  refusing a diff-less patch, then a real self-review of the real diff
  (correctness pass, missing tests fail) with the patch's own status
  confirmed untouched throughout.
- Tests: 291 backend (was 278) — 12 new (7 `selfReview.test.ts`, 5 new
  route tests), plus 1 new `aiRequestRepo.test.ts` case proving
  `patch_id`-keyed accounting stays independent per patch. 79 frontend
  (was 76) — 3 new. Full backend + frontend rebuilds green, both suites
  stable across repeated runs, `tsc --noEmit`/`tsc -b` and `vite build`
  clean.

## Phase 22 — Security Hardening (2026-08-19)
- Systematic verification pass over `docs/SECURITY.md`'s controls against
  the real implementation (not new product surface) plus a real
  windows-testrunner regression fix carried over from the prior Windows
  verification loop (see below).
- Windows testrunner fix, round 2: `backend/src/testrunner/run.ts`'s
  `runTests()` was passing `detached: true` unconditionally alongside
  piped stdio. On Windows, where `npm`/`pnpm`/`yarn` must be invoked
  through a `.cmd` wrapper (which `cross-spawn` does internally), that
  combination is a known Node.js footgun that broke stdio capture and
  left child processes alive past their own timeout, holding a file lock
  (`EBUSY` on test-fixture cleanup) — confirmed by real Windows `npm test`
  output showing empty `stdout_ref` and two timed-out
  `testrunner.test.ts` cases. Fixed by making it
  `detached: process.platform !== "win32"` — Windows never needed
  `detached` for tree-killing anyway, since `killProcessTree()`'s Windows
  branch already uses `taskkill /pid <pid> /t /f` against the OS's own
  tracked process tree. POSIX behavior unchanged. Verified stable
  (291/291, 2x) and clean-rebuilding in the Linux sandbox; transferred to
  the user's Windows checkout for real re-confirmation.
- Audited path sandboxing (§2) across every route: confirmed all
  filesystem access is scoped through `project.root_path` (validated once
  at registration, never re-supplied per request), and the one route
  where an AI-proposed path reaches disk
  (`generated-tests/:testId/write-and-run`) re-validates it via
  `resolveWithinRoot` before writing.
- Audited secret redaction (§4) across every AI-egress path: confirmed
  both context builders (`select.ts`, `selectContextForTestFailure.ts` —
  the only two in the product) redact all content before it's counted
  toward the token budget, with no bypass found; confirmed a generated
  patch's `diff_text` (used raw in Phase 21 self-review's prompt) can't
  carry an unredacted secret since it's the provider's own output,
  derived only from already-redacted context.
- Audited no-autonomous-execution (§6): confirmed both disk-writing/
  executing routes (`patches/:patchId/apply`,
  `generated-tests/:testId/write-and-run`) check the persisted
  approval-gate status server-side (not just hidden in the UI), and the
  test runner only ever executes its own fixed allow-listed command
  detection, never anything AI-supplied.
- Ran a real `npm audit` on both `backend/` and `frontend/` (§8). Found
  and fixed two genuine production-dependency vulnerabilities: `fastify`
  (high/critical, fixed by the 4→5 major bump) and `react-router-dom`
  (moderate open-redirect, fixed by the 6→7 major bump). Confirmed this
  product's real usage of each API is limited to long-stable surfaces
  (`app.get/post/patch/delete`, `request.body/params/query`,
  `reply.status` for Fastify; `BrowserRouter`/`Routes`/`Route`/`Link`/
  `NavLink`/`Outlet` for React Router) unaffected by either major
  version's breaking changes, before upgrading. Left the remaining
  `vite`/`vitest`/`esbuild` devDependency-only advisories as a
  documented, accepted risk rather than forcing a `vitest` 3→4 major bump
  across 34+9 test files for a dev-server-only, network-exposure-
  contingent issue — recorded in `docs/SECURITY.md` §9.
- Tests: 291 backend / 79 frontend — unchanged counts (this phase audited
  and upgraded, it did not add new product behavior), but both full
  suites reverified stable across 2 repeated runs each, and both
  `backend`'s and `frontend`'s production builds clean, on both the
  original dependency versions and again after the `fastify`/
  `react-router-dom` upgrades.

## Phase 23 — Performance Optimization (2026-08-19)
- Measured first, then fixed only what the numbers showed — see
  `docs/PERFORMANCE.md` for full methodology and data.
- Profiled the real discover→index→analyze pipeline directly (not
  through HTTP, to isolate compute from network overhead) against a
  real 3,672-file, 131MB corpus (this environment's own Node.js
  toolchain's `node_modules`, registered as a project root) — used
  because it's a large, real, messy directory tree, without needing
  network access to clone a public repository.
- Baseline full cycle: ~3.6s. `runAnalysis` dominated (~2.5-2.7s of
  it); per-rule profiling within `runAnalysis` found `missing-test-file`
  alone cost ~1.4s — over half.
- Root cause: `backend/src/analysis/rules/missingTests.ts`'s
  `hasCorrespondingTest()` re-scanned every path in the repo, from
  scratch, for every candidate source file — an O(files × allPaths)
  algorithm.
- Fix: precompute the set of naming-convention-covered base names once
  in a single O(allPaths) pass (`collectBaseNamesWithCorrespondingTest()`),
  turning each candidate's check into an O(1) `Set` lookup. Identical
  matching semantics — confirmed by the rule's finding count on the
  profiling corpus staying at 1,703 before and after, and all 5
  pre-existing correctness tests passing unmodified.
- Result on the same corpus: the rule dropped ~30x (1,369ms → 46ms),
  `runAnalysis` dropped ~2.6x (2,681ms → 1,015ms), and the full
  discover+index+analysis cycle dropped ~48% (3.6s → 1.9s).
- Added a real regression-guard test in `backend/test/analysis.test.ts`:
  builds a synthetic 1,500-file worst-case repo (every file a genuine
  naming-convention miss) and asserts the rule completes under a 400ms
  bound the old O(N²) code would have blown through at that scale —
  the fixed version measures single-digit milliseconds, confirmed via a
  standalone check.
- Profiled two other candidate costs and found no real problem to fix:
  discovery/indexing/analysis each independently walking the file tree
  (a deliberate freshness-over-speed tradeoff already documented in
  `analysis/context.ts`, and split across three separate real HTTP
  requests in the actual product flow, not compounding in one call) and
  three other rules' per-file regex passes (linear in file count/content
  size, not quadratic). Documented as accepted, not silently ignored —
  see `docs/PERFORMANCE.md`'s "known remaining costs" section.
- Tests: 292 backend (was 291) — 1 new perf-regression test. Frontend
  unchanged (no frontend work this phase), 79/79. Both suites reverified
  stable across 2 repeated runs, both builds clean.

## Phase 24 — Packaging (2026-08-19)
- Scoped from `docs/PRD.md` §3's "local-first" principle, since no
  existing doc named a specific desktop-shell/installer approach: made
  the concrete, low-risk step actually available — collapsing the
  dev-time split (separate Fastify API + separate Vite dev server,
  joined only by a dev proxy) into one process on one port a user can
  actually run — without deciding anything about Electron/Tauri/Docker/
  installers, which stays explicitly deferred to Phase 25
  ("Deployment / self-hosting"). See `docs/PACKAGING.md` for the full
  scope writeup and rationale.
- `buildApp()` (`backend/src/app.ts`) gained an optional `staticDir`:
  when set to a real built-frontend directory, registers
  `@fastify/static` plus an SPA-fallback 404 handler (a direct load or
  refresh of a client-side route like `/findings` re-serves
  `index.html`; an unmatched `/api/*` route still returns a real JSON
  404, never silently masked by the SPA fallback). Omitted/invalid
  `staticDir` leaves the backend exactly as API-only as it's always
  been — purely additive.
- `backend/scripts/copy-frontend.mjs` (plain Node `fs`, cross-platform,
  same reasoning as the existing `copy-migrations.mjs`) copies
  `frontend/dist` into `backend/dist/public` as part of `backend`'s own
  `npm run build`, degrading safely (a log message, not a build
  failure) when the frontend hasn't been built.
- `config.ts`'s new `staticDir` resolution defaults relative to the
  compiled module's own location (`import.meta.url`, not
  `process.cwd()`), overridable via `CODEBASE_ENGINEER_STATIC_DIR`, and
  only treated as real when `<dir>/index.html` actually exists —
  correctly resolves to `null` during the normal `vitest` run.
- 8 new backend tests (`staticServing.test.ts`): a real fixture
  "frontend build" on disk, real `app.inject()` requests through the
  real Fastify + `@fastify/static` pipeline, including a genuine
  path-traversal attempt confirming a file just outside the static root
  is never served. 300 backend tests total (was 292).
- Dogfooded fully end to end (not just via `app.inject()`): built the
  real frontend, ran the real backend build (which copied it in),
  started the real combined process (`node dist/server.js`) with a real
  temp data directory and a non-default port, and issued real `curl`
  requests confirming `GET /` served the real `index.html`, a real
  static asset under `/assets/` was served, `GET /findings` correctly
  fell back to `index.html`, `GET /api/v1/health` still returned the
  real API response, and `GET /api/v1/<nonexistent>` returned a real
  JSON 404.
- Dependency review carried over from Phase 22's pattern: added
  `@fastify/static@10.1.3` (compatible with the already-upgraded
  `fastify@5.12.1`), confirmed via a fresh `npm audit` that it
  introduced no new vulnerabilities.
- Updated `README.md`: replaced a long-stale "Phase 0/1 — project
  scaffold, no product features implemented yet" status line (this
  project is 24 of 26 phases complete) with an accurate summary
  pointing to `docs/FEATURE.md` as the source of truth, and added a
  "Running as a single process (packaged)" quick-start section.
- Tests: 300 backend (was 292) — 8 new. Frontend unchanged (no frontend
  code changes this phase), still 79/79. Both suites reverified stable
  across repeated runs, `tsc --noEmit`/`tsc -b` clean, both builds
  clean.

## Phase 25 — Deployment / Self-Hosting (2026-08-19)
- Scoped from `docs/PRD.md` §3/§7, same approach as Phase 24: this
  product has never been a multi-tenant hosted service, so "self-hosting"
  means running it on a machine you control, not a SaaS. Documented
  plainly (new `docs/DEPLOYMENT.md`) that there's no built-in
  authentication on the product's own `/api/v1/*` surface, and that
  exposing it beyond your own machine needs your own reverse proxy +
  auth — building one wasn't decided in this phase.
- **Docker path (built and run for real, not just written)**: a
  multi-stage root `Dockerfile` — Debian-based (not Alpine) so
  `better-sqlite3`'s native addon stays glibc-compatible when copied
  between build stages, builds the frontend then the backend (reusing
  Phase 24's `copy-frontend.mjs` unmodified), installs `git` as a real
  runtime dependency in the slim final image, runs as the base image's
  existing unprivileged `node` user. `docker-compose.yml` quick-start,
  validated with `docker compose config`.
- Actually built and exercised end to end in this project's own dev
  sandbox: hit a real network/TLS quirk specific to this sandboxed
  environment (an intercepting proxy with a self-signed cert that broke
  `npm ci` inside the container, and `apt-get` needing to bypass that
  same proxy for `deb.debian.org`) — solved for verification purposes
  only via a temporary, deleted-after-use Dockerfile variant with
  proxy/CA build args scoped narrowly to just the `npm ci` steps; the
  shipped `Dockerfile` itself has none of that sandbox-specific code, and
  was re-diffed against the tested variant to confirm they're
  structurally identical apart from those temporary lines. Once built: a
  real container started and logged serving the UI; real `curl` requests
  confirmed the health endpoint, the served UI, and `git --version`
  inside the running container; a real host directory (this project's own
  `backend/src`) was bind-mounted in and registered as a project through
  the real API; a real discover→index→analyze cycle ran inside the
  container and produced 61 real findings; the container was then
  destroyed and a fresh one started against the same named volume,
  confirming the registered project and its data survived a full
  container recreation, not just a process restart.
- **systemd path**: `deploy/codebase-engineer.service`, a real systemd
  **user** unit (never root) for running the packaged Node process
  directly on a Linux host without Docker. `systemd-analyze verify`
  caught a genuine bug in the first draft — a hardcoded `/usr/bin/node`
  path that doesn't exist in this dev sandbox's actual Node install
  location — fixed to `ExecStart=/usr/bin/env node dist/server.js` so it
  resolves via `PATH` instead, then reverified clean (0 errors). Could
  not verify a live-running service in this sandbox (no user D-Bus
  session available in a cloud dev container) — documented honestly in
  `docs/DEPLOYMENT.md` as verified-by-static-analysis, not
  verified-by-live-run, unlike the Docker path.
- No backend/frontend source changes this phase (pure deployment
  tooling + docs) — full test suites reverified unaffected regardless:
  300 backend / 79 frontend, both stable.
- README updated with a Docker self-hosting quick-start and a link to
  the new deployment doc.

## Phase 26 — Optional Monetization Architecture / Razorpay (2026-08-19)
- `docs/ROADMAP.md` had marked this phase P3, "Not Started — explicitly
  deferred" (unlike every other phase's plain "Not Started"), and
  `docs/PRD.md` §7 lists billing/payments as an explicit MVP non-goal.
  When "continue" arrived after Phase 25, this was flagged rather than
  auto-started: asked the user directly via `AskUserQuestion` whether to
  build monetization architecture now. The user answered "Yes, build it
  now" — that explicit confirmation is what authorized this phase.
- Built as real, working architecture per `docs/MONETIZATION.md`'s
  isolation principle, entirely opt-in: with no `RAZORPAY_KEY_ID`/
  `RAZORPAY_KEY_SECRET`/`RAZORPAY_WEBHOOK_SECRET` set (the default), every
  billing check is a no-op — usage stays unlimited and Free Mode/AI-Mode-
  with-a-user's-own-key are completely unaffected, verified by tests that
  hit real routes/functions with billing unconfigured.
- New additive migration `011_billing.sql`: `subscription` (single-row
  singleton — this product is single-instance/local-first per PRD §7, no
  multi-tenant/team billing) and `billing_webhook_event` (idempotent
  webhook redelivery tracking) tables. Migration-safety-verified against a
  simulated pre-existing database (migrations 001-010 applied, real
  `project`/`ai_request` rows inserted) opened via the real
  `openDatabase()`, confirming clean application with zero data loss.
- New self-contained `backend/src/billing/` module, imported by nothing in
  `analysis/`, `ai/`, `git/`, or `patch/`: `types.ts` (`TIER_LIMITS` — free
  = 50 AI operations/month, pro = unlimited), `config.ts`
  (`loadBillingConfig()`, `null` when unconfigured), `razorpayClient.ts`
  (real `fetch`-based Orders API client, HTTP Basic Auth, classified
  errors — `auth_error`/`rate_limited`/`unreachable` — tested against a
  real local HTTP double, not a mocked `fetch`), `webhookVerify.ts` (real
  HMAC-SHA256 signature verification via `crypto.timingSafeEqual`, with
  explicit length-mismatch handling since `timingSafeEqual` throws rather
  than returning false on mismatched lengths), `subscriptionRepo.ts`
  (get-or-create, activate, auto-expire-past-period, idempotent webhook
  recording), `usageLimiter.ts` (`checkAiOperationAllowed()` — the
  enforcement function, counting the current calendar month's `ai_request`
  rows and comparing against the tier's limit; reads the existing
  billing-agnostic `ai_request` table without needing any change to it).
- Enforcement wired into the single existing choke point:
  `routes/projects.ts`'s `resolveEnabledProvider()` — already the shared
  helper all 7 AI-spending routes call (explain, root-cause, fix-plan,
  generate-patch, generate-test, diagnose, self-review) — now calls
  `checkAiOperationAllowed()` first and returns a 402 before even
  resolving a provider once the caller is over their limit. Required zero
  changes to any of the 7 individual route handlers, since they already
  handle the generic `{error: {status, message}}` return shape.
- New `backend/src/routes/billing.ts`: `GET /api/v1/billing/status`,
  `POST /api/v1/billing/checkout` (creates a real Razorpay order),
  `POST /api/v1/billing/webhook` — registered inside its own encapsulated
  Fastify plugin context with a raw-`Buffer` content-type parser scoped
  only to that route, so signature verification runs against exact bytes
  while every other route keeps normal JSON parsing. Webhook handler
  activates a 30-day Pro period on `payment.captured`/`order.paid` events
  after verifying the signature and checking idempotency — explicitly a
  one-time-order-plus-webhook-verified-payment flow, not real Razorpay
  Subscriptions (recurring billing); documented as a natural next step if
  this is ever used for real, not attempted speculatively here.
- Frontend: `frontend/src/pages/Billing.tsx`, now served at `/settings`
  (replacing the placeholder), following `AiMode.tsx`'s established
  conventions (load pattern, `ApiError` handling, Tailwind styling,
  visibly-disabled/explained gating). Shows the not-configured state
  honestly, or the current tier/usage/limit with a visible warning once
  near or at the cap, plus an "Upgrade to Pro" button on the free tier
  that calls the real checkout endpoint. If `window.Razorpay` (the
  Checkout widget script) isn't loaded on the page, the UI honestly
  reports that the payment widget can't open — a real order is still
  created server-side — rather than fabricating a success state,
  consistent with this project's honest-degrade convention.
- Self-caught and fixed during authoring (before any test run): a dead
  `limitsFor()` function in `config.ts` using invalid CommonJS `require()`
  in this ESM/NodeNext project (deleted — `TIER_LIMITS` is already
  directly exported from `types.ts`); an unnecessary re-export in
  `routes/billing.ts`; a dead `??` fallback in `usageLimiter.ts` whose
  left side never actually returns `undefined`; an incorrect assumption in
  `usageLimiter.test.ts` about `replaceProjectFindings()`'s return value
  (it returns `void`, not created records — fixed by pre-generating the id
  and passing an `idFactory`, matching the existing
  `aiRequestRepo.test.ts` pattern).
- 27 new backend tests across `razorpayClient.test.ts` (5),
  `webhookVerify.test.ts` (8), `subscriptionRepo.test.ts` (5),
  `usageLimiter.test.ts` (6), and `routes.billing.test.ts` (8, including a
  real end-to-end test: registers a project, runs discover/index/
  analysis, enables a real local-double AI provider, makes 50 real
  successful `/explain` calls in a loop, then confirms a real 51st call
  returns 402 with a "limit reached" message) — 332 backend tests total
  (was 305). 7 new frontend tests in `Billing.test.tsx` covering
  not-configured, configured-under-limit, at-limit-warning, pro-tier, a
  real checkout call with the honest no-widget-script degrade path, a
  checkout-failure path, and a real Razorpay widget `open()` call when the
  script is present — 86 frontend tests total (was 79). Both suites
  reverified stable across 2 repeated runs; `tsc -b` and `vite build` both
  clean.
- `docs/MONETIZATION.md` rewritten from "Status: Not implemented" to
  document what's actually built; `docs/ROADMAP.md`'s Phase 26 row updated
  to "Done (tested)"; `docs/FEATURE.md` updated (a new Phase 26 row in the
  main table, and the "Future / Monetization" table's Razorpay row updated
  from "Not Started" to "Done (tested)").
