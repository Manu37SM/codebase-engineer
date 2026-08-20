# Going live — step-by-step

This walks through taking Codebase Engineer from "runs on my laptop" to
"reachable on the internet, safely." It's written as a sequence — do the
steps in order, don't skip the TLS/auth ones just because the app "already
works." Everything referenced here already exists in your codebase
(`docs/DEPLOYMENT.md`, `docs/AUTH.md`, `docs/SECURITY.md`) — this page is
the condensed, ordered checklist version.

Remember the core constraint this whole project is built around: this is
**self-hosted, single-instance software**, not a multi-tenant SaaS. "Going
live" means putting it on a machine you control (a VPS, a home server, a
work box) and reaching it over the internet — not standing up a hosted
product for other people's data.

## 0. Pick where it runs

Any of these work — pick based on what you already have:

- **A cheap VPS** (DigitalOcean, Hetzner, Linode, etc.) — cleanest option,
  a real public IP, no home-network NAT/port-forwarding to fight with.
- **A home server / NAS / spare machine** — works, but you'll need port
  forwarding or a tunnel (Cloudflare Tunnel avoids opening ports at all —
  worth considering if this is your route).
- **Your own machine, only reachable on your LAN/Tailscale** — if "live"
  just means "reachable from your phone/other devices, not the public
  internet," you can skip the domain/TLS/reverse-proxy steps below
  entirely and just bind `HOST=0.0.0.0` and use the machine's LAN IP or a
  Tailscale hostname. Everything from step 3 onward still applies if
  you're going public.

The rest of this guide assumes a public-facing deployment (VPS or home
server + a domain). If you're LAN-only, skip to step 6 (auth) and step 8
(backups) — you can skip TLS/reverse proxy, but still turn auth on if
anyone besides you can reach the machine.

## 1. Get a domain pointed at the machine

Buy/use a domain (or a subdomain, e.g. `ce.yourdomain.com`), and add an A
record (or AAAA for IPv6) pointing at your server's public IP. This can
take a few minutes to propagate.

## 2. Deploy the app itself

**Docker is the recommended path** — it's the one that's actually been
built, run, and verified end-to-end (see `docs/DEPLOYMENT.md`'s "What was
actually verified" section: real image build, real container, real
discover→index→analyze cycle, persistence confirmed across a container
recreation). The systemd path exists and passed static verification but
hasn't been dogfooded as a live running service — prefer Docker unless you
have a specific reason not to.

On the server:

```bash
git clone <your repo(s)> codebase-engineer && cd codebase-engineer
mkdir -p repos   # put/symlink the repositories you want to analyze here
docker compose -f deploy/docker-compose.yml --project-directory . up -d --build
```

Confirm it's actually running before moving on:

```bash
curl http://localhost:4000/api/v1/health
```

At this point the app is running but only reachable on `localhost` inside
that machine — that's intentional, the reverse proxy in step 4 is what
exposes it publicly. Don't open port 4000 directly to the internet.

## 3. Set the secrets it needs

Before anyone else can reach this, set real values (not the placeholders)
via a `backend/.env` file if running outside Docker, or as environment
variables passed into the container if using Docker/Compose (add an `env`
or `env_file` block to `deploy/docker-compose.yml`):

- `AUTH_TOKEN_ENCRYPTION_KEY` — only required if you turn on Google/GitHub
  OAuth (step 7), but generate and set it now regardless:
  `openssl rand -hex 32`. Never reuse the example value from
  `backend/.env.example`.
- `TRUST_PROXY=1` — set this once the reverse proxy (step 4) is in place,
  not before. See step 5.
- Anything else in `backend/.env.example` is genuinely optional — leave it
  unset unless you're specifically turning that feature on (Razorpay
  billing, Turnstile, OAuth).

## 4. Put a reverse proxy in front of it for TLS

Don't expose port 4000 straight to the internet — put something in front
that terminates HTTPS. **Caddy is the easiest option** here since it
handles Let's Encrypt certificates automatically with essentially zero
config:

```
# /etc/caddy/Caddyfile
ce.yourdomain.com {
    reverse_proxy localhost:4000
}
```

```bash
sudo systemctl reload caddy
```

Caddy forwards `X-Forwarded-Proto` automatically, so no extra config is
needed there. (If you'd rather use nginx or Traefik, both work fine — see
`docs/DEPLOYMENT.md`'s reverse-proxy section for the nginx header line
you'd need to add manually.)

Verify HTTPS actually works before continuing:

```bash
curl -I https://ce.yourdomain.com/api/v1/health
```

## 5. Tell the app to trust the proxy

Now that a real reverse proxy is terminating TLS, set `TRUST_PROXY=1` in
the backend's environment and restart the container:

```bash
docker compose -f deploy/docker-compose.yml --project-directory . up -d
```

This is what lets the session cookie's `secure` flag turn on correctly —
skip this and the login cookie will either not work over HTTPS, or (if you
forget the proxy is in front) accept a spoofable header from an untrusted
client. Don't set it before step 4 is actually in place.

## 6. Turn on authentication

This is the step that's easiest to accidentally skip — the app runs in
"open" mode (no login wall at all) until you register the first account.
**Register that first account now, before telling anyone else the URL**:

Go to `https://ce.yourdomain.com`, and register with an email + password
(8+ characters). The moment that account exists, every route except
`/api/v1/auth/*` and the health check starts requiring a session. There's
no toggle back to open mode short of deleting rows from the database
directly — so this is a one-way door, which is exactly what you want
before going public.

Optional at this point, not required to go live safely:
- **Google/GitHub OAuth sign-in** — set the four/three env vars from
  `docs/AUTH.md` §3 if you want "Sign in with Google/GitHub" instead of
  (or alongside) email/password.
- **Cloudflare Turnstile** — set `TURNSTILE_SECRET_KEY` (backend env var)
  and `VITE_TURNSTILE_SITE_KEY` (a Docker build ARG, not a runtime env
  var — see `deploy/docker-compose.yml`'s `build.args`, since Vite bakes
  it into the built JS) if you want bot protection on the login form on
  top of the IP-based rate limiting that's already built in. Changing
  either requires `docker compose ... up -d --build` (not just a restart)
  since the site key only takes effect at build time.

## 7. Firewall

Make sure only ports 80/443 (for the reverse proxy) and 22 (SSH) are open
to the internet on this machine — port 4000 (the app itself) should only
be reachable from `localhost`, since the reverse proxy is the only thing
that should ever talk to it directly:

```bash
sudo ufw allow 22
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
sudo ufw status
```

## 8. Set up backups

Everything that matters lives in the named Docker volume
(`codebase-engineer-data`, mounted at `/data`) — that's the SQLite
database with your registered projects, findings, and now user accounts.
Back it up on a schedule, not just once:

```bash
docker run --rm -v codebase-engineer-data:/data -v "$PWD":/backup debian \
  tar czf /backup/codebase-engineer-data-$(date +%F).tar.gz -C /data .
```

Put that in a daily cron job, and copy the resulting tarball somewhere off
that same machine (another server, S3, Backblaze — anywhere that survives
this machine dying).

## 9. Final pre-launch check

Before sharing the URL with anyone:

- [ ] `curl -I https://ce.yourdomain.com` shows a valid cert and, per the
      recent security audit, `x-content-type-options`,
      `x-frame-options`, `content-security-policy`, and
      `strict-transport-security` headers on the response.
- [ ] You can log in at the real URL with the account from step 6.
- [ ] `docker compose ... ps` shows the container as `Up`/healthy, and
      `restart: unless-stopped` (already set in `deploy/docker-compose.yml`)
      means it comes back after a server reboot.
- [ ] A backup tarball from step 8 actually exists and its cron job is
      scheduled.
- [ ] Port 4000 is NOT reachable from outside the machine
      (`curl http://YOUR_SERVER_IP:4000` from your own laptop should
      time out/refuse, not succeed).
- [ ] You've run `npm audit` in `backend/` and `frontend/` and reviewed
      what it reports — this sandbox has never had network access to check
      current advisories, so that's still on you to do once, from your own
      machine.

## 10. Day-to-day after launch

- **Upgrading:** `git pull && docker compose -f deploy/docker-compose.yml --project-directory . up -d --build` — migrations run automatically, the named volume (and your data) survives the rebuild.
- **Logs:** `docker compose -f deploy/docker-compose.yml logs -f`.
- **Known open items** (from the recent security audit, `docs/SECURITY.md`
  §10) worth revisiting as the app grows: account lockout is currently
  IP-based only, not per-account; there's no password-reset flow yet —
  check `docs/SECURITY.md` before assuming either of these is handled.
  (Turnstile's frontend widget is now wired in — see step 6 above; set
  both env vars and rebuild the image to turn it on.)
