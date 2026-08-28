# Codebase Engineer

A local-first developer platform for understanding, auditing, maintaining,
testing, reviewing, and improving real software repositories.

The core product works **without an AI API key** (Free Mode). AI-assisted
capabilities (explanations, root-cause analysis, fix planning, patch
generation, test generation, failure diagnosis, self-review) are an optional
layer called **AI Mode**, built behind a provider-agnostic interface, and
always require explicit human approval before any code is modified.

Status: **All 26 roadmap phases complete** — the full P0 (MVP) and P1 (AI
Mode) feature set is implemented and tested (repository discovery/indexing,
dashboard, architecture explorer, deterministic analysis, findings, git
analysis, test runner, dependency/security analysis, audit reports, and the
complete AI Mode workflow from context selection through self-review), plus
security-hardening, performance-optimization, packaging (this doc's "Running
the packaged app" section below), and deployment/self-hosting passes
(see "Self-hosting" below). Phase 26 adds an entirely opt-in monetization
architecture — inactive by default; every other feature works identically
with or without it configured.

## Documentation

This README is the project-level entry point. The former `docs/` set was
removed after launch; its history remains in this repo's git log.

Each subproject also has its own README with subproject-specific
run/build/test instructions: [`backend/README.md`](backend/README.md),
[`frontend/README.md`](frontend/README.md). This root README is the
project-level entry point; those cover day-to-day work inside each
subproject in more depth.

## Project layout

```
backend/    Node.js + TypeScript + Fastify API, SQLite persistence (own README, own git history)
frontend/   React + TypeScript + Vite + Tailwind CSS SPA (own README, own git history)
deploy/     Dockerfile, docker-compose.yml, and the systemd unit — deployment artifacts
```

`backend/` and `frontend/` are each tracked in their own git repository
(not the repo root's) — see "Version control layout" below.

## Getting started (development)

Requires Node.js 18+.

### Backend

```bash
cd backend
npm install
npm run build     # type-check + compile
npm run dev        # start in watch mode (http://localhost:4000)
npm test            # run backend tests
```

On startup the backend creates/updates a local SQLite database (see
`backend/src/config.ts` for the data directory) and applies migrations from
`backend/src/db/migrations/`.

### Frontend

```bash
cd frontend
npm install
npm run dev     # http://localhost:5173
npm run build    # production build
npm test          # run frontend tests
```

### Running as a single process (packaged)

For actually running the app day to day, rather than developing it, build
both and start the backend — it serves the built frontend itself on the
same port:

```bash
cd frontend && npm install && npm run build
cd ../backend && npm install && npm run build
npm start   # http://127.0.0.1:4000 — UI and API on one process, one port
```

Environment variables are documented in `backend/.env.example` and
`frontend/.env.example`.

### Self-hosting (Docker or systemd)

The Dockerfile and compose file live in `deploy/`, but the build context is
still the repo root — run this from the repo root:

```bash
docker compose -f deploy/docker-compose.yml --project-directory . up -d --build   # http://localhost:4000
```

`deploy/codebase-engineer.service` is the systemd unit for a non-Docker
install. Review `deploy/docker-compose.yml`'s `environment:` block before
exposing this beyond your own machine.

## Version control layout

This project intentionally uses **three separate git repositories**, not
one repo-wide history:

- `backend/` — its own repo (`backend/.git`), covering only backend
  source/tests.
- `frontend/` — its own repo (`frontend/.git`), covering only frontend
  source/tests.
- The repo root — its own repo, covering everything that isn't inside
  `backend/` or `frontend/`: `deploy/` (Dockerfile,
  compose file, systemd unit), and this README. The root repo's
  `.gitignore` excludes `backend/` and `frontend/` outright, since those
  are independently version-controlled — the root repo never sees inside
  them.

When committing a change, commit it into whichever of the three repos
actually owns the changed path — a backend source change goes into
`backend/`'s repo, a docs change goes into the root repo, and so on. There
is no single command that commits "everything" across all three; each is
managed independently (`git -C backend ...`, `git -C frontend ...`, or a
plain `git ...` from the repo root for root-level files).

## Principles

1. Local-first — your source code stays on your machine.
2. Free-first — the core product needs no AI key.
3. AI is optional — the app stays fully useful without it.
4. Human control — AI never silently modifies files; every change is a
   reviewable diff.
5. Provider independence — no hard dependency on any single AI vendor.
6. Security first — repositories are treated as untrusted input; secrets are
   detected and redacted; nothing AI-generated executes without approval.
