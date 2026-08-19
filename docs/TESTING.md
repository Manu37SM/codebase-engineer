# Testing Strategy — Codebase Engineer

## 1. Principles

- Real tests only. No superficial tests written purely to inflate coverage
  numbers.
- A feature is not "Done" in `/docs/FEATURE.md` until it has tests, the tests
  pass, the full relevant suite still passes, and the behavior was manually
  verified at least once.
- Deterministic analysis rules must be tested against fixture repositories with
  known expected findings (including true negatives — files that should *not*
  trigger a rule).

## 2. Tooling

- Backend: Vitest, run against `backend/src` and `backend/test`.
- Frontend: Vitest + React Testing Library, run against `frontend/src`.
- No end-to-end browser test framework is introduced in Phase 0; may be added
  later (e.g. Playwright) once core flows exist.

## 3. Required Coverage Areas (from product spec)

- Repository discovery
- Ignore handling (`.gitignore` + project rules)
- Language detection
- Indexing (file walk, metrics)
- Findings (rule engine, evidence integrity — no fabricated evidence)
- Git analysis
- Dependency analysis
- Security scanning (secret detection + redaction correctness)
- API endpoints (request validation, path-sandboxing enforcement)
- AI provider abstraction (adapter contract compliance, provider-agnostic
  business logic)
- Context selection (budgeting, explainability, exclusion correctness)
- Patch generation (diff correctness)
- Diff parsing
- Patch approval workflow (state machine: proposed → approved/rejected →
  applied/failed; illegal transitions rejected)
- Test execution (runner adapters, command allow-listing)
- Security boundaries (path traversal rejection, symlink escape rejection,
  secret non-leakage into AI context)
- Critical UI flows (navigation, finding → explain/root-cause/fix flow, patch
  review flow)

Each of these areas gets real test suites starting from the phase that
implements it (see `/docs/ROADMAP.md`); Phase 0 only ships scaffold-level smoke
tests to prove the toolchain works end to end.

## 4. Phase 0 Test Status

- `backend/test/health.test.ts` — smoke test asserting the Fastify app boots
  and `GET /api/v1/health` returns 200 with expected shape.
- `backend/test/db.test.ts` — smoke test asserting the SQLite migration runner
  applies migrations idempotently and `schema_migrations` is populated.
- `frontend/src/App.test.tsx` — smoke test asserting the app shell renders the
  top-level navigation sections without crashing.

These are intentionally minimal — they exist to prove the toolchain (build +
test runner + db init) works, not to validate product behavior, since no
product behavior exists yet.

## 5. Quality Gate

A feature is complete only when: implementation exists; tests exist; tests
pass; build passes; existing tests still pass; documentation is updated;
`/docs/FEATURE.md` is updated; security implications reviewed; manual
verification performed where appropriate. Never report a feature as
"implemented successfully" without having actually run and observed the
verification steps above.
