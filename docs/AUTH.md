# Authentication — Codebase Engineer

Local accounts, Google/GitHub OAuth sign-in, and optional Cloudflare
Turnstile bot protection. Everything in this document is **opt-in**: a
fresh install behaves exactly as every prior phase of this product always
has (no login wall anywhere) until you actually register the first
account.

## 1. Open mode vs. auth-required mode

The instance tracks how many accounts exist in its own `user` table:

- **Zero accounts (open/legacy mode, the default):** every route works
  with no session at all, same as before this feature existed. This is
  what a brand-new install looks like, and what every existing
  single-user setup keeps working like if you never register an account.
- **One or more accounts (auth-required mode):** the moment the first
  account is registered — local email/password, or the first successful
  Google/GitHub sign-in — every route under `/api/v1/*` except the auth
  routes themselves (`/api/v1/auth/*`) and the health check
  (`/api/v1/health`) starts requiring a valid session cookie. There is no
  toggle to go back to open mode short of deleting every row from the
  `user` table directly in the SQLite database.

This is deliberately one-directional and automatic — see
`backend/src/auth/guard.ts` for the actual check
(`countUsers(db) === 0`).

## 2. Local email/password accounts

Works with **zero configuration** — no environment variables required.

- `POST /api/v1/auth/register` — `{ email, password, displayName? }`.
  Password must be at least 8 characters. Sets a session cookie and
  returns the new (public) user record.
- `POST /api/v1/auth/login` — `{ email, password }`. Deliberately returns
  the identical error message ("Invalid email or password.") whether the
  email is unknown or the password is wrong, so failed attempts can't be
  used to enumerate registered emails.
- `POST /api/v1/auth/logout` — clears the session (both the cookie and
  the server-side session row).
- `GET /api/v1/auth/me` — `{ authRequired: boolean, user: PublicUser | null }`.
  The frontend uses `authRequired` to decide whether to show a login
  screen at all.

Passwords are hashed with Node's built-in `scrypt` (OWASP-recommended
parameters — see `backend/src/auth/password.ts`), never stored or logged
in the clear. Sessions are httpOnly cookies (`ce_session`) containing a
random 32-byte token; only the token's SHA-256 hash is ever persisted
(`backend/src/db/userRepo.ts`), so a stolen database backup alone can't be
used to forge a session.

## 3. Google / GitHub OAuth sign-in

Both are independent and optional — configure one, both, or neither.
Nothing in the local email/password flow depends on either being set up.

### Google

1. In [Google Cloud Console](https://console.cloud.google.com/), create an
   OAuth 2.0 Client ID of type **Web application**.
2. Add an authorized redirect URI matching `GOOGLE_REDIRECT_URI` below
   (default `http://localhost:4000/api/v1/auth/google/callback` — update
   the host/port to match your actual deployment).
3. Set in `backend/.env`:
   ```
   GOOGLE_CLIENT_ID=...
   GOOGLE_CLIENT_SECRET=...
   GOOGLE_REDIRECT_URI=http://localhost:4000/api/v1/auth/google/callback   # optional, this is the default
   AUTH_TOKEN_ENCRYPTION_KEY=<output of: openssl rand -hex 32>
   ```
4. Point a "Sign in with Google" link/button at `GET /api/v1/auth/google/start`
   — it redirects to Google, and the callback logs the browser in and
   redirects back to `/`.

### GitHub

1. In GitHub, go to **Settings → Developer settings → OAuth Apps → New
   OAuth App**.
2. Set the **Authorization callback URL** to match `GITHUB_REDIRECT_URI`
   below (default `http://localhost:4000/api/v1/auth/github/callback`).
3. Set in `backend/.env`:
   ```
   GITHUB_CLIENT_ID=...
   GITHUB_CLIENT_SECRET=...
   GITHUB_REDIRECT_URI=http://localhost:4000/api/v1/auth/github/callback   # optional, this is the default
   AUTH_TOKEN_ENCRYPTION_KEY=<output of: openssl rand -hex 32>              # same key Google uses, if both are configured
   ```
4. Point a "Sign in with GitHub" link/button at `GET /api/v1/auth/github/start`.

GitHub sign-in requests the `repo` scope (in addition to `read:user
user:email`) — not just for authentication, but because a later feature
(repo browsing/clone-to-register) uses the same access token to list and
clone the signed-in user's repositories on their own behalf. The token is
encrypted at rest (AES-256-GCM, see `backend/src/auth/crypto.ts`) using
`AUTH_TOKEN_ENCRYPTION_KEY` and is never used for anything the UI didn't
explicitly trigger.

### Linking behavior

Signing in with Google/GitHub links to an existing local account if one
already exists with the same (verified) email address — so registering
with `alice@example.com`/a password and later clicking "Sign in with
Google" using the same address lands in the same account, not a
duplicate. If no matching account exists, a new passwordless account is
created automatically.

### `AUTH_TOKEN_ENCRYPTION_KEY`

Required once *either* OAuth provider is configured — the backend throws
loudly (rather than silently using a fallback key) if an OAuth callback
runs without it set. Generate a real random value:

```bash
openssl rand -hex 32
```

Do not reuse the placeholder value from `backend/.env.example`.

## 4. Cloudflare Turnstile (bot protection)

Optional, and independent of everything else above — leave
`TURNSTILE_SECRET_KEY` unset and register/login work exactly as described
above with no CAPTCHA step at all.

To turn it on:

1. Create a Turnstile widget in the [Cloudflare dashboard](https://dash.cloudflare.com/) and note its **site key** and **secret key**.
2. Backend: set `TURNSTILE_SECRET_KEY` in `backend/.env`.
3. Frontend: set `VITE_TURNSTILE_SITE_KEY` in `frontend/.env` (this one is
   meant to be public — it's embedded in the page the browser loads,
   unlike every other value on this page).
4. The register/login forms render the Turnstile widget and include its
   token as `turnstileToken` in the request body; the backend verifies it
   server-side against Cloudflare's `siteverify` endpoint
   (`backend/src/auth/turnstile.ts`) and rejects the request with 400 if
   verification fails or the token is missing.

Once configured, verification **fails closed**: a missing token, or a
network error reaching Cloudflare, both result in the request being
rejected rather than silently allowed through.

## 5. Going live behind a reverse proxy

See `docs/DEPLOYMENT.md`'s "Going live behind a reverse proxy" section —
in short, set `TRUST_PROXY=1` once (and only once) this process sits
behind a real reverse proxy terminating TLS, so the session cookie's
`secure` flag turns on correctly.

## 6. What this is not

Per the user's explicit architecture decision (see `docs/PRD.md` §3/§7):
this is single-instance, local-account/OAuth authentication for a
self-hosted install — not a hosted multi-tenant SaaS backend. There is no
per-user server-side project storage; every registered project's
`root_path` still has to exist on the machine this process runs on
(`docs/SECURITY.md` §2), regardless of which account is logged in.
AI-assisted file changes always operate on that same local filesystem —
authentication controls who can reach the app, not where its data lives.
