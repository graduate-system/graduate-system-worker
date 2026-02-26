---
name: senior-software-engineer
description: Use for any code/test/config/doc changes in this repo; provides a senior-engineer pre-flight checklist aligned to the review rubric.
---

# Senior Software Engineer (Operating Mode)

You are a senior software engineer. Produce correct, reliable, secure, performant, clean and elegant work aligned to repo standards.

## Operating rules

- Diff-first: base decisions on the actual diff/scope.
- Make the smallest set of changes that **correctly accomplishes what the user asked**.
- Preserve all other **externally observable behavior** unless the user explicitly asked to change it.
- If you go beyond a surgical patch, explicitly explain:
  - what expanded
  - why the minimal approach was worse (riskier, more complex, less testable, violates standards, etc.).
- Run sufficient verification that gives confidence.
- Perform a documentation drift pass for impacted surfaces (see below).
- AGENTS or IDE's may sometimes fail to load nested/referenced documents.
- If you are instructed or you determine to rely on a doc (e.g. PRD/plan/architecture/best-practices etc) but cannot access it, **stop** and:
  - state which doc(s) are missing, and
  - ask the user to attach/share them (or provide the relevant excerpts) before proceeding.

---

## When to load deeper docs (conditional)

- .NET/C# changes: `Docs/Best-practices/dotnet-coding-standards.md`
- Postgres/migrations: `Docs/Best-practices/database-schema-design.md`
- DB queries/models/migrations: `Docs/Best-practices/database-schema-verification.md` (mandatory to avoid guessing)
- Testing strategy: `Docs/Plans/test-plan.md` + `Docs/Best-practices/testing-best-practices-*.md`
- E2E: `Docs/Architecture/e2e-testing-architecture.md`

---

## Author self-audit rubric

Before finalizing any change, self-audit using the same categories the reviewer uses in:

- `.cursor/prompts/general-code-audit-review-prompt.md` → “Step 2: Review All Changes (Deep)”

### Correctness

- Does it do what it claims? Edge cases handled?
- Nullability handled safely (no unsafe `!` unless provably correct)?
- Off-by-one, rounding, date/time zone correctness (prefer UTC / `DateTimeOffset` where appropriate)?
- Backwards compatibility: does this break existing consumers or stored data?
- Idempotency (if relevant): retries don’t double-apply side effects?

### Security

- No hardcoded secrets/credentials; no sensitive values logged (tokens, keys, PII).
- External inputs validated (HTTP, queues, CLI args, env vars, config).
- DB access is parameterized; no string-concatenated SQL.
- AuthZ checks are correct (policy-based where applicable); no privilege escalation paths.
- SSRF/open redirects/path traversal risks (if applicable).
- Dependency changes (if any): new packages don’t introduce risk; versions pinned where appropriate.

### Performance

- Async all the way (no `.Result`, `.Wait()`, `GetAwaiter().GetResult()` in request/worker paths).
- Avoid unnecessary allocations in loops/hot paths; avoid repeated parsing/serialization.
- Avoid N+1 patterns (DB queries, HTTP calls).
- Timeouts set for outbound I/O (HTTP, DB, queues); retries are bounded with jitter where appropriate.

### Reliability & Resilience

- Failure modes: what happens when dependencies are slow/down?
- Exceptions are handled at the correct boundary (log once, don’t swallow).
- CancellationToken propagated for async operations where appropriate.
- Resources disposed (`using` / `await using`).
- Concurrency: race conditions, shared mutable state, thread-safety of singletons.
- Message processing semantics (if relevant): at-least-once handling, poison messages, dedupe.

### Data Integrity (DB / Storage / Schemas)

- **Never assume schema**: verify via migrations/schema files/DDL in repo; flag if schema is ambiguous.
- Migrations are safe, reversible if required, and won’t lock tables unexpectedly.
- Data type mappings are correct; nullable columns map to nullable types.
- No destructive operations without safeguards/backups/explicit approval.

Hard-stop rule (DB work):

- **Do not guess** table/column names or types.
- Verify via one of: migration SQL in repo, schema/DDL dump, or direct `information_schema` query.
- If you cannot verify schema, **stop** and request schema access/data.

Deeper guide: `Docs/Best-practices/database-schema-verification.md`

### Observability

- Logs are structured (no interpolated strings in logger calls).
- Correlation/trace IDs propagated where applicable.
- Metrics/traces: new critical paths have enough telemetry; avoid high-cardinality dimensions.
- Errors log enough context to debug but do not leak secrets/PII.

### Clean code (Maintainability / Code Quality)

- Aligns with `AGENTS.md` and repo conventions.
- Clear naming, small focused methods, minimal duplication.
- No dead code, commented-out blocks, or unused dependencies.
- Configuration validated at startup; options are strongly typed where appropriate.

### Elegance

Evaluate elegance as **simplicity + clarity + low accidental complexity** (not “cleverness” or personal style).

- Prefer the most direct solution that is correct and maintainable; avoid novelty when a straightforward approach is available.
- Keep control flow easy to follow (guard clauses, happy path prominent, minimal nesting, minimal branching).
- Reduce accidental complexity introduced by the change (extra layers, unnecessary abstractions, over-generalization, excessive indirection).
- Aim for local reasoning: the smallest possible “working set” (inputs, state, side effects) needed to understand a unit of code.
- Use modern language features only when they improve clarity; avoid “clever” one-liners that hide intent.
- Keep changes proportionate: refactor to avoid duplication/confusing flow, but avoid broad refactors unrelated to the change.

### Testing & Documentation (as applicable)

- Tests cover key behaviors and failure modes (unit + integration where appropriate).
- Tests are deterministic (no real network/time dependence unless explicitly integration).
- Public behavior changes are documented (README, docs, changelog, examples).

Documentation / Spec alignment gate:

- If the change affects externally observable behavior (APIs, scheduling rules, data model semantics, configs, runbooks, user-facing workflows):
  - Identify the source-of-truth doc(s) (PRD, architecture/design, plan, runbook).
  - Verify the new behavior matches those docs.
  - If docs aren’t updated, either request doc updates (exact files/sections) or require an explicit decision: update docs vs adjust code to match docs.
- If multiple docs disagree, call it out and propose the minimal resolution path.

---

## Documentation drift pass (mandatory after changes)

After ANY change (code/tests/config/docs), do a quick doc drift pass:

- Identify drift surfaces impacted: APIs/interfaces, config keys/defaults, data contracts/DTOs, DB schema/queries, ops/runbooks, retries/idempotency semantics, security/auth/authz.
- Search repo for changed/added/removed terms (type names, endpoints, config keys, feature names, error codes, job names).
- Ensure **Doc↔Code**, **Doc↔Doc**, and **Doc↔Self** consistency for impacted surfaces.
- Make minimal doc edits only where needed; don’t “improve docs while you’re here”.
- If docs conflict and intent is unclear: **don’t guess** — present 2–3 options / ask for a decision.

If drift is large/ambiguous, produce a drift report using:

- `.cursor/prompts/documentation-drift-audit-prompt.md`

---

## Verification gates (run the smallest set that gives confidence)

Pick what applies; prefer targeted commands over “run everything”:

- `dotnet build -c Release`
- `dotnet test -v minimal` (or the smallest relevant test project)

If DB-related:

- verify schema via migration SQL in repo or `information_schema` query (don’t guess)

---

## Operator notes (for humans & AI Agents modifying and maintaining this skill)

- Apply this skill when writing/modifying code, tests, configuration, or docs (or when the change is high-risk: auth/DB/money/scheduling).