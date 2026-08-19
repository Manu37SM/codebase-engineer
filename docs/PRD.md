# Product Requirements Document — Codebase Engineer

Status: Phase 0 (Draft)
Owner: Engineering
Last updated: 2026-08-18

## 1. Problem

Developers need a tool that helps them understand, audit, maintain, test, review, and
improve real software repositories. Most existing tools in this space are either:

- Pure AI chat wrappers that require an API key and send entire repositories to a
  third-party model, or
- Narrow single-purpose linters/scanners with no unified workflow.

Codebase Engineer aims to be a professional developer product that is useful on day
one with zero AI configuration, and becomes more powerful — not fundamentally
different — when AI Mode is enabled.

## 2. Vision

Codebase Engineer is a local-first developer platform for repository understanding,
static analysis, testing, and (optionally) AI-assisted root-cause analysis and code
change proposals. See `/docs/ARCHITECTURE.md` for system design and
`/docs/AI_MODE.md` for the AI capability layer.

## 3. Product Principles

1. **Local-first** — source code stays on the developer's machine; no requirement to
   upload a whole repository to a remote server.
2. **Free-first** — the core product (discovery, indexing, dashboard, static
   analysis, git analysis, test running, dependency/security analysis, audit
   reports) works with zero AI configuration.
3. **AI is optional** — the app remains fully useful with no AI provider configured,
   an unavailable provider, or an exhausted usage limit.
4. **Human control** — AI never silently modifies source code; every AI-proposed
   change is shown as a diff and requires explicit human approval before it touches
   disk.
5. **Provider independence** — AI providers are pluggable via an `AIProvider`
   interface; no vendor lock-in in business logic.
6. **Security first** — repositories are treated as untrusted input; secrets are
   detected and redacted; no AI-generated shell command executes without explicit
   approval.

## 4. Users

- Individual developers auditing or onboarding onto an unfamiliar repository.
- Tech leads wanting a repeatable audit/health report for a codebase.
- Teams wanting AI-assisted (but human-approved) root cause analysis and fix
  proposals without granting an agent unrestricted write/execute access.

## 5. Modes

### Free Mode (no AI key required)
Repository discovery, repository index, dashboard, architecture explorer,
deterministic static analysis, findings, git analysis, test runner, dependency
analysis, security analysis (deterministic), audit report.

### AI Mode (optional, pluggable provider)
Finding explanation, root-cause analysis, fix planning, patch (diff) generation,
AI test generation, failure diagnosis, AI self-review, AI-assisted refactoring
proposals. All gated behind human approval before any file is modified.

Full feature boundary table: see `/docs/AI_MODE.md` §1.

## 6. MVP Definition (P0)

The MVP ships when all of the following exist, are tested, and function against a
real local repository:

- Repository discovery (root/git/language/framework/build-system detection) —
  initial support: Java, JavaScript, TypeScript, Maven, npm, pnpm, yarn.
- Repository indexing respecting `.gitignore` and standard excludes.
- Dashboard showing repo metadata, LOC, dependency and framework summary.
- Architecture explorer (aggregated module/dependency view, not a raw node dump).
- Deterministic static analysis engine producing structured `Finding` records.
- Findings list/detail UI.
- Git analysis (branch, status, recent commits, diff stats).
- Test runner (Maven, npm scripts, Vitest) with captured stdout/stderr and pass/
  fail/skip counts.
- Dependency analysis (direct deps, counts, duplicate versions where detectable).
- Security analysis (deterministic secret/config smell detection with redaction).
- Audit report (composite health score + detected facts vs. recommendations).
- AI provider abstraction (interface + at least one adapter, e.g. an
  OpenAI-compatible adapter, wired but **not required** to run) with visible
  provider/model/usage/status.
- AI context selection engine (budgeted, explainable selection — implemented even
  if only exercised by tests in Phase 0/1).
- AI explanation, AI fix planning, and diff generation flows — behind the
  provider abstraction, not implemented with hardcoded vendor calls.
- Human approval workflow for any AI-proposed change (UI + API gate).

Full phase breakdown: see `/docs/ROADMAP.md`. Feature-by-feature status: see
`/docs/FEATURE.md`.

## 7. Explicit Non-Goals for MVP

- No autonomous coding agent (no unattended multi-step code edits).
- No billing/payments (Razorpay or otherwise).
- No execution of AI-generated shell commands without explicit per-command
  approval.
- No multi-tenant/team features.
- No cloud-hosted repository storage.

## 8. Success Criteria (Phase 0)

- Frontend builds (`npm run build` in `frontend/`).
- Backend builds and starts (`npm run build` / `npm run dev` in `backend/`).
- SQLite database initializes via migrations on backend startup.
- `npm test` (Vitest) executes and passes in both `frontend/` and `backend/`.
- Documentation set exists and accurately reflects what was actually built (no
  aspirational claims of completeness).
