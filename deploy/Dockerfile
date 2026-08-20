# Codebase Engineer — self-hosted container image (Phase 25)
#
# Builds the frontend and backend, then produces a slim runtime image that
# runs the same single combined process Phase 24's packaging introduced
# (`node dist/server.js`, serving the built frontend + API on one port).
# See docs/DEPLOYMENT.md for how to run this, what a self-hosted deployment
# needs to provide (a persistent volume for the SQLite database, and a bind
# mount for whatever repositories you want to register as project roots —
# this app only ever reads paths that exist inside its own container
# filesystem, so a repo on the host is invisible to it unless mounted in),
# and the explicit non-goals (no built-in auth/TLS — see that doc).
#
# Uses a Debian-based image (not Alpine) so `better-sqlite3`'s native
# addon, compiled once in the `backend-build` stage, is binary-compatible
# with the same glibc runtime in the final stage — copying prebuilt
# `node_modules` across stages avoids a second native compile in the
# runtime image while keeping the runtime image free of the build
# toolchain (python3/make/g++) `better-sqlite3`'s `npm ci` needs.

# ---- Stage 1: build the frontend ----
FROM node:22-bookworm-slim AS frontend-build
WORKDIR /app/frontend
# Vite env vars are baked into the built JS at build time, not read at
# container runtime — so the Turnstile site key (meant to be public, see
# docs/AUTH.md §4) has to come in as a build ARG, not a compose
# `environment:` entry. Empty by default, which keeps Turnstile off
# (frontend/src/lib/turnstile.ts treats an unset/empty key as "disabled").
ARG VITE_TURNSTILE_SITE_KEY=""
ENV VITE_TURNSTILE_SITE_KEY=$VITE_TURNSTILE_SITE_KEY
COPY frontend/package.json frontend/package-lock.json ./
RUN npm ci
COPY frontend/ ./
RUN npm run build

# ---- Stage 2: build the backend (and pull the built frontend into it) ----
FROM node:22-bookworm-slim AS backend-build
# python3/make/g++ are needed only here, for better-sqlite3's native build
# during `npm ci` — never shipped in the final runtime image.
RUN apt-get update \
    && apt-get install -y --no-install-recommends python3 make g++ \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /app/backend
COPY backend/package.json backend/package-lock.json ./
RUN npm ci
COPY backend/ ./
# Placed at ../frontend/dist relative to /app/backend, matching exactly
# what scripts/copy-frontend.mjs (Phase 24) resolves and expects — so
# `npm run build`'s existing copy-frontend step (no Docker-specific
# branch) picks it up the same way a local `npm run build` would.
COPY --from=frontend-build /app/frontend/dist /app/frontend/dist
RUN npm run build
RUN npm prune --omit=dev

# ---- Stage 3: runtime ----
FROM node:22-bookworm-slim AS runtime
# `git` is a real runtime dependency, not just a build-time one: Git
# analysis, diff review, and patch apply (docs/FEATURE.md Phases 8/17/18)
# all shell out to the real `git` binary via execFileSync. Java/Maven
# projects' test runner support (Phase 9) is a documented gap in this
# base image — see docs/DEPLOYMENT.md; extend this image with a JDK +
# Maven if you need to run Maven-based test suites from inside the
# container.
RUN apt-get update \
    && apt-get install -y --no-install-recommends git \
    && rm -rf /var/lib/apt/lists/*

# The node:*-bookworm-slim base image already ships an unprivileged
# `node` user (uid/gid 1000) — reused here rather than creating a new one,
# since a fresh `useradd --uid 1000` collides with it.
WORKDIR /app
COPY --from=backend-build /app/backend/dist ./dist
COPY --from=backend-build /app/backend/node_modules ./node_modules
COPY --from=backend-build /app/backend/package.json ./package.json

ENV HOST=0.0.0.0
ENV PORT=4000
ENV CODEBASE_ENGINEER_DATA_DIR=/data
RUN mkdir -p /data && chown -R node:node /data /app
VOLUME ["/data"]
EXPOSE 4000
USER node

CMD ["node", "dist/server.js"]
