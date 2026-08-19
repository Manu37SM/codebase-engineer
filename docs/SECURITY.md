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

Do not treat this document as evidence that a control is active until its
corresponding phase is marked "Done (tested)" in `/docs/FEATURE.md`.
