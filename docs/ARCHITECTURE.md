# Architecture — Codebase Engineer

## 1. System Boundaries

Codebase Engineer is a locally-run application with three logical components:

1. **Frontend** — React + TypeScript + Vite + Tailwind CSS SPA. Talks only to the
   local backend over HTTP (`localhost`). No direct filesystem or AI provider
   access.
2. **Backend** — Node.js + TypeScript + Fastify API server. Owns all filesystem
   access (sandboxed to a selected repository root), all Git operations, the
   deterministic analysis engine, the SQLite database, the test runner, and the
   AI provider abstraction (backend is the only component allowed to hold AI
   provider credentials).
3. **SQLite database** — single-file, local, holds indexed repository metadata,
   findings, analysis runs, test runs, AI request/response logs (metadata only —
   see `/docs/SECURITY.md`), patches, patch reviews, audit reports, and provider
   configuration.

No component sends repository source code anywhere except: (a) to the local
filesystem/Git, and (b) optionally, a minimally-scoped context bundle to a
user-configured AI provider when AI Mode is explicitly invoked by the user for a
specific finding/action.

```
┌─────────────┐      HTTP (localhost)      ┌──────────────────────────┐
│  Frontend    │ ─────────────────────────▶ │  Backend (Fastify)       │
│  React/Vite  │ ◀───────────────────────── │                          │
└─────────────┘                             │  ┌────────────────────┐ │
                                             │  │ Discovery/Indexer   │ │
                                             │  ├────────────────────┤ │
                                             │  │ Analysis Engine     │ │
                                             │  ├────────────────────┤ │
                                             │  │ Git Service         │ │
                                             │  ├────────────────────┤ │
                                             │  │ Test Runner         │ │
                                             │  ├────────────────────┤ │
                                             │  │ AI Provider Layer   │─┼──▶ (optional) external
                                             │  │  + Context Engine   │ │      AI provider
                                             │  ├────────────────────┤ │
                                             │  │ Patch/Diff Service  │ │
                                             │  └────────────────────┘ │
                                             │           │              │
                                             │           ▼              │
                                             │      SQLite (local)      │
                                             └──────────────────────────┘
                                                        │
                                                        ▼
                                          Local filesystem (sandboxed to
                                          the selected repository root)
```

## 2. Repository Structure

```
codebase-engineer/
├── docs/                    Product & engineering documentation (source of truth)
├── progress/                Living progress tracking (CURRENT/COMPLETED/BLOCKED)
├── backend/
│   ├── src/
│   │   ├── app.ts           Fastify app factory
│   │   ├── server.ts        Entrypoint
│   │   ├── config.ts        Environment/config loading
│   │   ├── db/
│   │   │   ├── index.ts     SQLite connection + migration runner
│   │   │   └── migrations/  Numbered SQL migrations
│   │   ├── discovery/       Repository discovery (language/framework/build detect)
│   │   ├── indexer/         File walking, .gitignore handling, metrics
│   │   ├── analysis/        Deterministic static analysis rules + Finding model
│   │   ├── git/             Git analysis service
│   │   ├── testing/         Test runner adapters (maven/npm/vitest)
│   │   ├── dependencies/    Dependency analysis
│   │   ├── security/        Deterministic security scanning + redaction
│   │   ├── audit/           Audit report composition
│   │   ├── ai/
│   │   │   ├── provider/    AIProvider interface + adapters
│   │   │   ├── context/     Context selection & budgeting engine
│   │   │   └── workflows/   Explanation, root-cause, fix-plan, patch generation
│   │   ├── patch/           Diff generation, patch review/approval, apply
│   │   ├── routes/          Fastify route modules (one per resource)
│   │   └── services/        Cross-cutting services
│   └── test/                Vitest test suites (mirrors src/ structure)
├── frontend/
│   ├── src/
│   │   ├── pages/           Dashboard, Repositories, Architecture, Findings,
│   │   │                    Changes, Tests, Audit, AIMode, Settings
│   │   ├── components/      Shared UI components
│   │   ├── lib/             API client, types
│   │   └── App.tsx / main.tsx
│   └── test/                React Testing Library + Vitest
└── README.md
```

## 3. Database Model (SQLite)

Entities (see migration `001_init.sql` for authoritative schema):

- `project` — a registered local repository (id, name, root_path, created_at).
- `repository_snapshot` — result of a discovery run (languages, frameworks,
  build_system, package_managers, git_branch, working_tree_status, indexed_at).
- `file` — indexed file (project_id, relative_path, language, loc, size_bytes,
  is_test, is_generated, content_hash).
- `finding` — deterministic or AI-derived finding (id, project_id, rule_id,
  severity, category, file_path, line_start, line_end, evidence, explanation,
  recommendation, source [`deterministic`|`ai`], created_at).
- `analysis_run` — a static analysis execution (id, project_id, started_at,
  finished_at, status, findings_count).
- `test_run` — a test execution (id, project_id, framework, command, exit_code,
  duration_ms, passed, failed, skipped, stdout_ref, stderr_ref, started_at).
- `ai_request` — metadata about an AI call (id, project_id, provider, model,
  operation_type, estimated_tokens, status, created_at). **Never stores raw
  secrets; prompt/response bodies are stored only if the user opts in, and are
  redacted per `/docs/SECURITY.md`.**
- `ai_response` — linked response metadata (id, ai_request_id, estimated_tokens,
  latency_ms, success).
- `patch` — a proposed code change (id, project_id, finding_id nullable,
  description, diff_text, status [`proposed`|`approved`|`rejected`|`applied`|
  `failed`], created_at).
- `patch_review` — human review decision on a patch (id, patch_id, decision,
  reviewer_note, decided_at).
- `audit_report` — composite report (id, project_id, scores JSON, findings
  summary JSON, created_at).
- `provider_configuration` — AI provider config (id, name, kind, base_url,
  model, api_key_ref [never the raw key — see Security], enabled, created_at).

Migrations are plain numbered SQL files under `backend/src/db/migrations/`,
applied in order by a minimal migration runner at startup, tracked in a
`schema_migrations` table.

## 4. API Architecture

Fastify, versioned under `/api/v1`. Resource-oriented routes, one module per
resource under `backend/src/routes/`:

- `GET  /api/v1/health`
- `POST /api/v1/projects` / `GET /api/v1/projects` / `GET /api/v1/projects/:id`
- `POST /api/v1/projects/:id/discover`
- `POST /api/v1/projects/:id/index`
- `GET  /api/v1/projects/:id/dashboard`
- `GET  /api/v1/projects/:id/architecture`
- `POST /api/v1/projects/:id/analysis` / `GET .../findings`
- `GET  /api/v1/projects/:id/git`
- `POST /api/v1/projects/:id/tests/run` / `GET .../tests/:runId`
- `GET  /api/v1/projects/:id/dependencies`
- `GET  /api/v1/projects/:id/security`
- `GET  /api/v1/projects/:id/audit` / `GET .../audit/export` (Markdown) —
  implemented as `GET`, not `POST` as originally sketched here: it's a
  read-only aggregation of other endpoints' data (Phase 11), not an
  action with side effects, matching this API's convention for
  computed-live endpoints (`/git`, `/security`, `/dependencies`).
- `GET/POST/PATCH/DELETE /api/v1/ai/providers[/:id]` (configure/list/update/
  remove; never returns raw keys — Phase 12), plus `POST .../:id/check-status`
  and `GET .../:id/models` for live connectivity checks
- `POST /api/v1/ai/findings/:id/explain`
- `POST /api/v1/ai/findings/:id/root-cause`
- `POST /api/v1/ai/findings/:id/fix-plan`
- `POST /api/v1/ai/patches` (generate proposed patch — returns diff, does not
  write files)
- `POST /api/v1/patches/:id/approve` / `POST /api/v1/patches/:id/reject`
- `POST /api/v1/patches/:id/apply` (only callable after `approved`)

All project-scoped routes validate that resolved paths remain inside the
project's registered root (see `/docs/SECURITY.md` §Path Traversal).

## 5. Frontend Architecture

Single-page app, React Router-based navigation matching the nine top-level
sections (Dashboard, Repositories, Architecture, Findings, Changes, Tests,
Audit, AI Mode, Settings). AI surfaces are contextual (an "Explain" / "Root
Cause" / "Create Fix" action attached to a Finding; a "Diagnose" action attached
to a failed test; a review screen attached to a Patch) rather than a standalone
chat window. State/data-fetching via a thin typed API client in
`frontend/src/lib/`.

## 6. Analysis Engine Architecture

The deterministic analysis engine (`backend/src/analysis/`) is a rule-based
pipeline: each rule receives the indexed file set + parsed structure (where a
parser is available) and yields zero or more `Finding` objects with a stable
`rule_id`, severity, category, file/line evidence, explanation, and
recommendation. Rules must never fabricate evidence — a rule that cannot cite a
concrete file/line/pattern match does not fire. Language-aware rules use
Tree-sitter grammars where available (starting with Java, JavaScript,
TypeScript); rules that only need text/line heuristics (e.g. large-file, TODO
count) do not require a parser and work for any language.

## 7. AI Provider Architecture

See `/docs/AI_MODE.md` §2 for the full `AIProvider` interface design and adapter
list. Summary: business logic (workflows in `backend/src/ai/workflows/`) depends
only on the `AIProvider` interface, never on a concrete SDK. Adapters
(OpenAI-compatible, Anthropic-compatible, Ollama/local) implement that interface
and are selected at runtime via `provider_configuration`.

## 8. AI Context Architecture

See `/docs/AI_MODE.md` §3. Summary: a `ContextSelector` takes a target (a
Finding, a failed TestRun, etc.) and returns a bounded, explained `ContextBundle`
(selected files/snippets + reasons + excluded-because-out-of-budget list),
never the whole repository, subject to a configurable token budget.

## 9. Security Model

See `/docs/SECURITY.md` for the full model (path sandboxing, secret detection/
redaction, no auto-exec of AI-generated commands, human-approval gate for all
writes).

## 10. Testing Strategy

See `/docs/TESTING.md`.
