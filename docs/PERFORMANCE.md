# Performance — Codebase Engineer

This document records real profiling methodology and results for the
deterministic Free Mode pipeline (discovery → indexing → analysis →
architecture), and the optimizations made as a result. It is a living
record, not a promise — numbers here are from one real corpus on one
machine, not a guaranteed SLA on every repository.

## Methodology

Profiling is done by calling the real backend functions directly (not
through HTTP, to isolate compute from network/Fastify overhead) against a
real, large, pre-existing directory tree — not a synthetic fixture — using
`process.hrtime.bigint()`/`performance.now()` around each call. The corpus
used for the Phase 23 pass was `node_modules` of this environment's own
Node.js toolchain: 3,672 indexed files (1,905 JavaScript, 552 TypeScript,
plus non-source files), 131MB on disk. It was registered as a project root
directly (not scanned as a nested `node_modules` inside another project,
which the file walker's default excludes would skip) specifically to get a
large, real, messy, real-world file tree without needing network access to
clone a public repository.

Whole-pipeline timing (`discoverRepository` → `indexRepository` →
`runAnalysis`, sequential, matching the real "Scan" button's flow) was
measured first to find where the wall-clock time actually goes, then
per-rule timing inside `runAnalysis` (the dominant phase) to find which
rule specifically was expensive, before writing any optimization — the
methodology used throughout is "measure first, do not guess."

## Phase 23 findings and fix

Baseline (before any Phase 23 change), full discover+index+analysis cycle
against the corpus above: **~3.6 seconds**. Breakdown:

| Step | Time |
|---|---|
| `discoverRepository` | ~210-700ms |
| `indexRepository` | ~570-890ms |
| `runAnalysis` | **~2,460-2,680ms** |

`runAnalysis` was clearly the dominant cost, so it was profiled per-rule:

| Rule | Time |
|---|---|
| `missing-test-file` | **~1,369ms** |
| `secret-smell` | ~180-200ms |
| `large-function` | ~200-215ms |
| `permissive-cors` | ~110ms |
| `disabled-tls-verification` | ~55-60ms |
| `todo-fixme-density` | ~75-80ms |
| `large-file` | <1ms |
| `env-file-committed` | ~1-3ms |

`missing-test-file` alone accounted for over half of `runAnalysis`'s total
time, and roughly 38% of the entire scan cycle. Reading
`backend/src/analysis/rules/missingTests.ts` found the cause:
`hasCorrespondingTest()` re-scanned every path in the repo, from scratch,
for every candidate source file being checked — an O(files × allPaths)
algorithm. On a repo with ~3,600 files where most files have no matching
test, this is close to its worst case.

**Fix**: replaced the per-candidate linear scan with a single O(allPaths)
precomputation pass (`collectBaseNamesWithCorrespondingTest()`) that builds
the set of source base names already covered by a naming-convention test
match, once, before the main loop — turning each candidate file's check
into an O(1) `Set.has()` lookup. This is a pure algorithmic change with
identical matching semantics (verified by: the rule's real finding count on
the profiling corpus was unchanged at 1,703 findings before and after; all
5 existing `missing-test-file` correctness tests in
`backend/test/analysis.test.ts` continued to pass unmodified).

Result: `missing-test-file` dropped from ~1,369ms to **~46ms** (about 30x),
`runAnalysis` dropped from ~2,680ms to **~1,015ms**, and the full
discover+index+analysis cycle dropped from ~3.6s to **~1.9s** (about 48%
faster end to end) on the same corpus.

A regression guard test (`"scales roughly linearly with repo size, not
quadratically"` in `backend/test/analysis.test.ts`) constructs a synthetic
1,500-file repo shaped to be the worst case for the old algorithm (every
file is a genuine miss, so the old code always scanned to exhaustion) and
asserts the rule completes in well under the time the old O(N²) behavior
would take at that scale — the fixed version measures single-digit
milliseconds against a 400ms bound, leaving generous headroom for slower CI
hardware while still catching a real algorithmic regression.

## Known remaining costs (not changed in Phase 23 — documented, not fixed)

- **Discovery, indexing, and analysis each walk the repository's file tree
  independently** (`walkRepository` is called once per phase). This is a
  deliberate design choice, not an oversight: `docs/ARCHITECTURE.md` and
  `backend/src/analysis/context.ts`'s own doc comment explain that analysis
  re-walks fresh off disk (rather than reading Phase 3's persisted file
  index) specifically so a finding's evidence is always traceable to
  current file content, not a potentially-stale index. In the real product
  these three phases are also three separate HTTP requests (discover, then
  index, then analyze — see the "Scan" button's flow in Phase 4's
  `FEATURE.md` entry), not one combined call, so the duplicate walk cost
  is spread across requests rather than compounding within one. Removing
  this duplication would require a shared, invalidation-aware cache across
  requests — real complexity weighed against a comparatively modest
  remaining cost (the three walks together are a few hundred milliseconds
  of the ~1.9s total post-fix). Left as-is; revisit if profiling on a real
  user repository shows this dominating in practice.
- Several other rules (`secret-smell`, `large-function`,
  `permissive-cors`) each do their own regex pass over every file's text.
  These were profiled and are linear in file count and total content size,
  not quadratic — no algorithmic issue found, just proportional-to-input
  cost. Not optimized further in this pass.

## Re-running this profiling

The profiling scripts used for this pass were throwaway (`profile.mjs`,
`profile2.mjs` in `backend/`, deleted after use — not part of the shipped
product). To reproduce: build the backend (`npm run build`), then import
`dist/discovery/index.js`, `dist/indexer/index.js`, `dist/analysis/index.js`
(or individual rule modules under `dist/analysis/rules/` for per-rule
timing) and call them directly against a real, large directory of your
choosing, timing each call with `process.hrtime.bigint()` or
`performance.now()`.
