# Deployment / Self-Hosting — Codebase Engineer

Phase 25. Two verified ways to run Codebase Engineer somewhere other than
your own dev machine's `npm run dev`/`npm start`: a Docker image, and a
systemd user service for running the packaged Node process directly on a
Linux host. Both build on Phase 24's packaging (a single process serving
the API and the built UI on one port) — this phase doesn't change that
process, only how it's started and kept running.

## Before you deploy anywhere: what this product is and isn't

Per `docs/PRD.md` §3 ("local-first") and §7 (explicit non-goals: no
multi-tenant/team features, no cloud-hosted repository storage), Codebase
Engineer has never been designed as a multi-tenant hosted service. "Self-
hosting" here means **running it on a machine you control, for yourself**
(or your own team, trusting each other the way you'd trust anyone with
shell access to a shared dev box) — not standing up a public SaaS.
Concretely, that has real security consequences documented once here
rather than repeated in every section below:

- **Built-in authentication is opt-in, not on by default.** As of the auth
  system added after Phase 25 (see `docs/AUTH.md`), the instance runs in
  "open" mode — no login wall on any route, the same behavior this product
  has always had — until you actually register the first account (email +
  password, or Google/GitHub OAuth). The moment one account exists, every
  route under `/api/v1/*` except the auth routes themselves and the health
  check starts requiring a valid session. There is no way to go back to
  open mode short of deleting every row from the `user` table. The default
  `HOST=127.0.0.1` still keeps the dev server loopback-only regardless of
  auth mode, which matters most while you're still in open mode. If you
  deploy this anywhere reachable beyond your own machine, register an
  account (turning on the login wall) AND put a reverse proxy in front of
  it that terminates TLS — see "Going live behind a reverse proxy" below;
  relying on the built-in auth alone, over plain HTTP, on a network you
  don't fully trust, still leaves the session cookie and login form
  readable to anyone on the network path.
- **A project's `root_path` must exist inside wherever the process is
  running.** This was true before this phase (`docs/SECURITY.md` §2) and
  doesn't change here — it just becomes more visible in a container,
  where "inside" means "bind-mounted into the container's filesystem". A
  repository on your host machine is invisible to a containerized
  Codebase Engineer until you mount it in; see the Docker section below.

## Option A: Docker (recommended for most self-hosting)

A real multi-stage `deploy/Dockerfile` builds the frontend, then the
backend (which copies the built frontend into itself via Phase 24's
existing `copy-frontend.mjs` — no Docker-specific build logic needed
beyond copying files at the right relative paths between build stages),
then produces a slim Debian-based runtime image (~Node 22, not Alpine —
see the Dockerfile's own comments for why: `better-sqlite3`'s native
addon needs to be glibc-compatible with the runtime, and copying prebuilt
`node_modules` between same-base-image stages avoids a second native
compile in the final image). `git` is installed in the runtime image
(Git analysis, diff review, and patch apply all shell out to a real `git`
binary — see `docs/FEATURE.md` Phases 8/17/18), and so is `python3` +
`pytest`, so the built-in test runner can run pytest-based Python
projects out of the box (it only provides the `pytest` binary itself — a
project's own test dependencies beyond the stdlib still need to be
importable, same limitation the npm-script/vitest path has for a missing
`node_modules`). A JDK/Maven, Go, and Ruby are **not** installed, so the
test runner's Maven, Go, and RSpec support (Phase 9) won't work against
those projects inside this image unless you extend it with your own
`apt-get install default-jdk maven` / `golang-go` / `ruby-full` layer. If
a test command can't even be launched because its runtime is missing, the
Tests page now shows the real "command not found" error in the raw output
panel instead of a blank exit code.

The Dockerfile and compose file live in `deploy/` (alongside the systemd
unit), but the Docker **build context is still the repo root** — every
command below is meant to be run from the repo root, not from inside
`deploy/`, so the `COPY backend/ ...`/`COPY frontend/ ...` paths inside
the Dockerfile keep resolving the same way they always have.

### Quick start

```bash
git clone <this repo> && cd codebase-engineer
mkdir -p repos   # put (or symlink) the repositories you want to analyze in here
docker compose -f deploy/docker-compose.yml --project-directory . up -d --build
```

Open `http://localhost:4000`. To register a repository, use its
**in-container** path — anything under `./repos` at the repo root is
visible inside the container at `/repos/<same-relative-path>` (see
`deploy/docker-compose.yml`'s bind mount; `--project-directory .` is what
makes that `./repos` resolve against the repo root rather than
`deploy/`). A repository living somewhere else on your host needs its own
bind mount added to `deploy/docker-compose.yml` (or the equivalent
`docker run -v` flag if not using Compose).

Without Compose:

```bash
docker build -f deploy/Dockerfile -t codebase-engineer .
docker run -d --name codebase-engineer \
  -p 4000:4000 \
  -v codebase-engineer-data:/data \
  -v /path/to/your/repos:/repos:ro \
  codebase-engineer
```

`CODEBASE_ENGINEER_DATA_DIR` defaults to `/data` inside the image (see the
`Dockerfile`'s `ENV`), so the named volume above is where the SQLite
database lives — back it up with `docker run --rm -v
codebase-engineer-data:/data -v "$PWD":/backup debian tar czf
/backup/codebase-engineer-data.tar.gz -C /data .` (or your platform's
usual named-volume backup approach).

### Upgrading

```bash
git pull
docker compose -f deploy/docker-compose.yml --project-directory . up -d --build
```

Migrations run automatically against the existing database on the next
container start (`openDatabase()`'s existing migration runner, unchanged
by this phase) — the same behavior this product has always had, just now
inside a container instead of a bare `node dist/server.js`. The named
volume is untouched by a rebuild, so your registered projects and analysis
history survive.

### What was actually verified (not just written)

This was built and run for real in this project's own development
sandbox, not just written and assumed to work: `docker build` produced a
real image; a real container started and logged
`Serving built frontend from /app/dist/public`; real `curl` requests
confirmed the health endpoint, the served UI, and that `git --version`
works inside the running container; a real host directory (this project's
own `backend/src`) was bind-mounted in and registered as a project through
the real API, and a real discover → index → analyze cycle ran inside the
container and produced real findings (61, on that directory) — not a
mocked response. The container was then removed and a fresh one started
against the same named volume, and the registered project was still there,
confirming the persistence story actually holds across a container
recreation, not just a process restart. `deploy/docker-compose.yml` was
validated with `docker compose -f deploy/docker-compose.yml --project-directory . config`.
(This verification predates the Dockerfile/compose file's move into
`deploy/` — re-verified by inspection that the move is context/path-only,
since `build.context`/`dockerfile` in the compose file and `-f`/build
context in the raw `docker build` command were updated to compensate; the
Dockerfile's own `COPY` instructions are unchanged and still resolve
against the repo-root build context exactly as before.)

## Option B: systemd user service (Linux, no Docker)

For running the packaged Node process directly on a Linux host you already
manage, without a container. `deploy/codebase-engineer.service` is a real
systemd **user** unit (runs as your own user, not root — this app never
needs elevated privileges, only read/write access to whatever repositories
you register and its own data directory):

```bash
cd backend && npm install && npm run build   # and build+copy the frontend per docs/PACKAGING.md
mkdir -p ~/.config/systemd/user
cp ../deploy/codebase-engineer.service ~/.config/systemd/user/
# Edit WorkingDirectory in the copied file to your real checkout path.
systemctl --user daemon-reload
systemctl --user enable --now codebase-engineer
loginctl enable-linger "$USER"   # keeps it running after you log out
```

Check status/logs with `systemctl --user status codebase-engineer` and
`journalctl --user -u codebase-engineer -f`. The unit uses
`ExecStart=/usr/bin/env node dist/server.js` (resolves via `PATH` rather
than a hardcoded Node install location, since that varies a lot between a
system package, nvm, and other version managers) — if status shows
`node: not found`, add an `Environment=PATH=...` line pointing at wherever
your own `which node` reports; the unit file's own comments explain this.

**What was actually verified**: `systemd-analyze verify` was run against
the real unit file and passed cleanly (0 errors) after fixing exactly this
kind of real, caught issue — the first draft hardcoded `/usr/bin/node`,
which `systemd-analyze verify` correctly flagged as "not executable" in
this development sandbox (Node is installed elsewhere here), which is
exactly the real-world portability problem the `env node` fix addresses.
Actually starting a user-session systemd service could not be verified
inside this development sandbox (no user D-Bus session/login manager
available in a cloud dev container), so the "runs and stays up" behavior
relies on `systemd-analyze verify`'s syntax/semantics check plus the fact
that `node dist/server.js` run directly (not through systemd) has been
exercised repeatedly throughout this project — not a full live-service
dogfood the way the Docker path got. Treat this path as verified-by-static-
analysis-plus-component-testing, not verified-by-live-run, and report back
if `systemctl --user enable --now` behaves unexpectedly on a real machine.

## Going live behind a reverse proxy

Once you've registered an account (turning on the login wall — see
`docs/AUTH.md`) and want to reach this instance from somewhere other than
`localhost`/your own LAN, put a real reverse proxy (Caddy, nginx, Traefik,
Cloudflare Tunnel, etc.) in front of the Node process to terminate TLS.
Two things need to be true for the session cookie to behave correctly once
you do:

1. **The proxy must forward `X-Forwarded-Proto: https`** (and normally
   `X-Forwarded-For`) to this process — every proxy listed above does this
   by default or with one config line (e.g. Caddy's `reverse_proxy` does it
   automatically; nginx needs `proxy_set_header X-Forwarded-Proto $scheme;`).
2. **Set `TRUST_PROXY=1`** in this process's environment (see
   `backend/.env.example`). This tells Fastify to trust those forwarded
   headers — without it, the app has no way to know the proxy terminated
   TLS on the client's behalf, since this Node process itself always only
   ever speaks plain HTTP to the proxy.

With both of those in place, the auth session cookie's `secure` flag turns
on automatically (it's derived from the actual request protocol, not
hardcoded — see `setSessionCookie` in `backend/src/routes/auth.ts`), so the
cookie is only ever sent back over HTTPS once you're actually running over
HTTPS. Skip `TRUST_PROXY` (the default) for local dev or any setup where
this process is reached directly and there's no proxy to trust — turning it
on without a real proxy in front would let any client spoof its own
`X-Forwarded-*` headers.

This is exactly the same "reverse proxy handles the public TLS/network
edge, the app itself stays a plain local process" shape called out
generally in the section above, now with the specific two settings the
built-in auth system needs to cooperate with it correctly.

## Explicit non-goals for Phase 25

Consistent with `docs/PRD.md` §7 and the scoping note carried over from
Phase 24 (`docs/PACKAGING.md`'s "scope" section): no managed/hosted
multi-tenant offering (the auth system added later is single-instance,
local-account/OAuth login — not a per-user SaaS backend; see
`docs/AUTH.md`), no auto-updater, no native OS installer
(`.exe`/`.dmg`/`.deb`) or desktop shell (Electron/Tauri) — none of those
were decided here, and none should be inferred from what shipped in this
phase.
