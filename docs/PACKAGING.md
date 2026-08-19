# Packaging — Codebase Engineer

Phase 24. This describes how to build and run Codebase Engineer as a single
local process, and what "packaging" does and doesn't mean for this product
at this phase.

## Scope: what Phase 24 is, and isn't

Per `docs/PRD.md` §3's "local-first" principle, this product has never been
a hosted, multi-tenant service — it runs on the developer's own machine
against their own local SQLite database (`docs/SECURITY.md` §7). Phase 24
("Packaging") is about collapsing the development-time split — a Fastify
API process and a separate Vite dev server, each on their own port, wired
together only by a dev-time proxy — into **one process on one port** that a
user can actually run day to day, without changing anything about the
local-first architecture itself.

Phase 24 does NOT include (deliberately deferred to Phase 25,
"Deployment / self-hosting", which is a separate, later phase for a
reason): a native installer (`.exe`/`.dmg`/`.deb`), an auto-updater, a
desktop shell (Electron/Tauri or similar — no decision has been made on
this, and none should be inferred from anything shipped in this phase),
Docker packaging, or a hosted/remote deployment story. Those are real,
separate decisions with their own tradeoffs (bundle size, auto-update
security, code-signing, OS-specific installer tooling) that deserve their
own scoping pass rather than being assumed here.

## What changed

1. **The backend can now serve the built frontend.**
   `backend/src/app.ts`'s `buildApp()` takes an optional `staticDir`. When
   set to a real directory containing a built frontend (`index.html` +
   assets), the backend registers `@fastify/static` to serve it, plus an
   SPA-fallback 404 handler so a direct load or refresh of a client-side
   route (e.g. `/findings`) still returns `index.html` rather than a
   generic file-not-found. An unmatched `/api/*` route still returns a
   real JSON 404 — never silently served the frontend's `index.html`,
   which would turn a genuine "this API route doesn't exist" bug into a
   confusing blank page instead of a clear error. When `staticDir` is
   omitted or the directory doesn't have an `index.html`, the backend is
   exactly as API-only as it has always been — this is purely additive,
   confirmed by the full pre-existing test suite passing unmodified plus
   a new dedicated `backend/test/staticServing.test.ts` (8 tests) that
   builds a real fixture "frontend build" on disk and issues real
   `app.inject()` HTTP requests through the real Fastify + `@fastify/static`
   pipeline — including a genuine path-traversal attempt confirming a
   file just outside the static root is never served.

2. **The backend build copies the frontend's build output into itself.**
   `backend/scripts/copy-frontend.mjs` (plain Node.js `fs` calls, not a
   shell `cp -r` — the same cross-platform reasoning as the existing
   `copy-migrations.mjs`, since a POSIX-only shell pipeline breaks
   `npm run build` on Windows) copies `frontend/dist` into
   `backend/dist/public` as part of `backend`'s own `npm run build`. If
   the frontend hasn't been built yet, this step logs a message and exits
   cleanly rather than failing the backend build — packaging is additive,
   never required, consistent with `staticDir` being optional at the
   `buildApp()` level.

3. **`backend/src/config.ts` resolves `staticDir` automatically.**
   Defaults to `<the directory this compiled file lives in>/public`
   (via `import.meta.url`, not `process.cwd()`, so it works regardless of
   which directory the process is started from), overridable via
   `CODEBASE_ENGINEER_STATIC_DIR`. Only treated as real if
   `<staticDir>/index.html` actually exists — during the normal `vitest`
   run (which executes from `src/`, where no `public/` directory exists)
   this correctly resolves to `null`, so tests are unaffected.

## Running the packaged app

```bash
# One-time: build the frontend, then the backend (which copies the
# frontend's build output into itself).
cd frontend && npm install && npm run build
cd ../backend && npm install && npm run build

# Start the single combined process:
npm start
# -> Codebase Engineer backend listening on http://127.0.0.1:4000 (db: ~/.codebase-engineer/codebase-engineer.db)
# -> Serving built frontend from <repo>/backend/dist/public
```

Open `http://127.0.0.1:4000` — the same process serves the UI and the API,
on the same port, with no separate dev server or proxy involved.

Configuration (all via environment variables, unchanged from before this
phase except where noted):

| Variable | Default | Meaning |
|---|---|---|
| `PORT` | `4000` | Port the combined server listens on |
| `HOST` | `127.0.0.1` | Bind address — stays loopback-only by default, consistent with "local-first, not a hosted service" |
| `CODEBASE_ENGINEER_DATA_DIR` | `~/.codebase-engineer` | Where the local SQLite database lives |
| `CODEBASE_ENGINEER_STATIC_DIR` | `<backend>/dist/public` | New in Phase 24 — override where the backend looks for a built frontend to serve, if not the default co-located location |

This was dogfooded end to end (not just unit-tested): the real frontend was
built, the real backend build copied it in, the real combined process was
started (`node dist/server.js`) with a real temp data directory and a
non-default port, and real `curl` requests confirmed `GET /` serves the
real built `index.html`, a real static JS asset under `/assets/` is
served, `GET /findings` (a client-side-only route) correctly falls back to
`index.html` rather than 404ing, `GET /api/v1/health` still returns the
real API response, and `GET /api/v1/<nonexistent>` returns a real JSON 404
rather than the SPA fallback.

## Development mode is unchanged

For active development, the existing two-process setup (`npm run dev` in
`backend/` on port 4000, `npm run dev` in `frontend/` on port 5173 with
Vite's dev-time `/api` proxy to port 4000 — see `frontend/vite.config.ts`)
still works exactly as it did before this phase and remains the recommended
way to iterate on the frontend with hot module reload. The combined
single-process mode above is for actually *running* the app day to day, not
for developing it.
