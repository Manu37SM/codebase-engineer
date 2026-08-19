# AI Mode — Design

AI Mode is an optional capability layer. Free Mode never depends on it.

## 1. Free vs AI Mode Feature Boundary

| Capability | Free Mode | AI Mode |
|---|---|---|
| Repository discovery/indexing | ✅ | — |
| Dashboard / architecture explorer | ✅ | — |
| Deterministic static analysis & findings | ✅ | — |
| Git analysis | ✅ | — |
| Test running | ✅ | — |
| Dependency & security analysis (deterministic) | ✅ | — |
| Audit report (deterministic scores) | ✅ | — |
| Finding explanation (why it matters, likely cause) | — | ✅ |
| Root-cause analysis (evidence vs. inference) | — | ✅ |
| Fix planning (structured 7-part plan) | — | ✅ |
| Code change proposals (diff, human-approved) | — | ✅ |
| AI-generated tests (reviewed & executed, not trusted on compile alone) | — | ✅ |
| Failure diagnosis after a modification | — | ✅ |
| AI self-review of its own generated changes | — | ✅ |
| Refactoring proposals | — | ✅ |

If no AI provider is configured, all AI-Mode UI affordances are visibly
disabled (not hidden) with an explanation, and every Free Mode feature keeps
working normally.

## 2. AI Provider Architecture

```ts
interface AIProvider {
  readonly id: string;            // e.g. "openai-compatible", "anthropic-compatible", "ollama"
  readonly displayName: string;
  listModels(): Promise<AIModelInfo[]>;
  complete(request: AICompletionRequest): Promise<AICompletionResult>;
  estimateTokens(text: string): number;
  checkStatus(): Promise<AIProviderStatus>; // reachable / auth ok / rate-limited / down
}
```

- Business logic in `backend/src/ai/workflows/` depends only on this interface
  — never a concrete vendor SDK.
- Adapters live in `backend/src/ai/provider/adapters/` (planned: OpenAI-
  compatible, Anthropic-compatible, Ollama/local). Only the adapter module may
  import a vendor SDK/HTTP client.
- `provider_configuration` (DB table) stores which adapter + model + base URL
  is active; the API key itself is referenced indirectly (`api_key_ref`) and
  never returned to the frontend.
- The UI surfaces provider, model, usage, and status (reachable/error/rate-
  limited) per `/docs/ARCHITECTURE.md` §7.

## 3. AI Context Engine

Given a target (a `Finding`, a failed `TestRun`, a refactor request), the
`ContextSelector` builds a `ContextBundle`:

```ts
interface ContextBundle {
  targetId: string;
  budgetTokens: number;
  selected: { path: string; reason: string; tokens: number }[];
  excluded: { path: string; reason: string }[];
  totalTokens: number;
}
```

Selection order (stops once budget is spent): the directly affected file →
directly relevant methods/functions → imported symbols used at the finding
site → known callers (where statically determinable) → associated test file(s)
→ relevant config → relevant Git diff hunk. Every included and excluded item
carries a human-readable reason, shown in the UI (per the "Context budget"
example in the product spec). Content passes through the secret-redaction layer
(`/docs/SECURITY.md` §4) before being counted toward the budget or sent.

## 4. AI Workflow (human-approval gated)

```
Finding → AI Context Selection → AI Root Cause Analysis → Fix Plan →
Human Approval → Patch Generation → Diff Review → Human Approval →
Apply Patch → Run Tests → (if failure) AI Diagnosis → Proposed Fix →
Review → Final Verification → Audit Trail
```

Both approval gates (before patch generation proceeds to affect anything, and
before a patch is applied to disk) are enforced server-side by the `patch` /
`patch_review` state machine (`/docs/ARCHITECTURE.md` §3), not just in the UI.

## 5. Fix Plan Structure

Every AI fix plan has exactly these seven sections, each required:

1. Problem
2. Root cause
3. Files affected
4. Proposed changes
5. Risks
6. Required tests
7. Validation strategy

## 6. AI Self-Review Checklist

After generating a modification, before presenting it for human review, the
self-review workflow checks: correctness, scope creep, regressions, security,
missing tests, unnecessary complexity, and consistency with existing
architecture. Self-review output is shown alongside the diff, not used to
auto-approve.

## 7. Cost Control (accounting only — no billing in MVP)

Tracked per AI call in `ai_request`/`ai_response`: requests, estimated tokens,
provider, model, operation type, timestamp, success/failure. Configurable
limits (e.g. a monthly operation cap for a "free AI trial") are enforced by
checking accumulated usage before issuing a new request — implemented as a
simple counter/limit check, no payment processing. See
`/docs/MONETIZATION.md`.

## 8. Phase 0 Status

None of the above is implemented yet. This document defines the target design
for Phases 12–21. Phase 0 delivers documentation and a scaffold only.
