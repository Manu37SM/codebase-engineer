# Architecture Decision Records

## ADR-001: Local-first, two-layer product (Free Mode / AI Mode)

**Status:** Accepted (Phase 0)

**Context:** The product must be useful without any AI API key, and AI must be
strictly optional and provider-agnostic.

**Decision:** Split the system into a deterministic core (discovery, indexing,
static analysis, git, tests, dependencies, security, audit) that has zero
dependency on any AI provider, and an AI layer that sits behind an `AIProvider`
interface and is invoked only on explicit user action per finding/target.

**Consequences:** Slightly more upfront architecture work (interface + context
engine) but avoids vendor lock-in and keeps the product valuable with zero
external dependencies.

## ADR-002: SQLite for local persistence

**Status:** Accepted (Phase 0)

**Context:** Need simple, zero-ops local persistence for a single-machine
developer tool.

**Decision:** Use SQLite (via `better-sqlite3`) with hand-written numbered SQL
migrations and a minimal migration runner, rather than a heavier ORM/migration
framework, to keep the schema transparent and dependency footprint small.

**Consequences:** Migrations are manual SQL (more explicit, less magic); no
built-in schema-diffing tool. Acceptable for MVP scope; revisit if schema
complexity grows significantly.

## ADR-003: Fastify for the backend API

**Status:** Accepted (Phase 0)

**Decision:** Fastify + TypeScript, per the specified stack. Chosen over
Express for built-in schema validation hooks and lower overhead, which matters
for an API that will handle potentially large file-index payloads.

## ADR-004: React + Vite + Tailwind for the frontend

**Status:** Accepted (Phase 0)

**Decision:** Per specified stack. Vite for fast local dev iteration, Tailwind
for consistent utility-based styling suited to a dense developer-tool UI (see
`/docs/ARCHITECTURE.md` §5 and product design principles in the master spec).

## ADR-005: No autonomous agent execution before foundations are stable

**Status:** Accepted (Phase 0)

**Decision:** Human approval is the default and only mode for any AI-proposed
code change through at least Phase 21 (AI self-review). Full autonomous
multi-step execution is explicitly out of scope until the deterministic core,
AI provider abstraction, context engine, patch/diff/approval workflow, and
self-review are all implemented and stable.

## ADR-006: Diff-based patch model, never direct file mutation by AI

**Status:** Accepted (Phase 0)

**Decision:** AI code-change workflows always produce a `patch` record with a
unified diff (`diff_text`). Applying a patch to disk is a separate, explicit,
human-gated operation (`POST /api/v1/patches/:id/apply`, only valid after
`approved`). The AI workflow code path itself never has direct file-write
access.

## ADR-007: Payments/Razorpay explicitly deferred

**Status:** Accepted (Phase 0)

**Decision:** No billing/payment code in MVP. Internal usage-accounting data
model (requests, estimated tokens, provider, model, operation type, timestamp,
success/failure) is built starting at Phase 12/13 so that future monetization
(Phase 26) has data to build on, but billing logic itself stays isolated and
unimplemented until the core product is proven useful. See
`/docs/MONETIZATION.md`.
