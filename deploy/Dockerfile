

FROM node:26-bookworm-slim AS frontend-build
WORKDIR /app/frontend

ARG VITE_TURNSTILE_SITE_KEY=""
ENV VITE_TURNSTILE_SITE_KEY=$VITE_TURNSTILE_SITE_KEY
COPY frontend/package.json frontend/package-lock.json ./
RUN npm ci
COPY frontend/ ./
RUN npm run build

FROM node:26-bookworm-slim AS backend-build

RUN apt-get update \
    && apt-get install -y --no-install-recommends python3 make g++ \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /app/backend
COPY backend/package.json backend/package-lock.json ./
RUN npm ci
COPY backend/ ./

COPY --from=frontend-build /app/frontend/dist /app/frontend/dist
RUN npm run build
RUN npm prune --omit=dev

FROM node:26-bookworm-slim AS runtime

RUN apt-get update \
    && apt-get install -y --no-install-recommends git ca-certificates python3 python3-pip \
    && update-ca-certificates \
    && pip3 install --no-cache-dir --break-system-packages pytest \
    && rm -rf /var/lib/apt/lists/*

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
