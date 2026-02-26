# fix-merged

A **merged** audit report from `/review-parallel` or `/review-branch-parallel` produced consolidated findings; your job is to **validate** each reported issue and then **fix only the issues that are real**.

The source of truth for what to address is:

- The **latest** merged report from a parallel review run, stored under the run directory: `Output/CodeReview/RUN_*/AUDIT_REPORT_*_MERGED.json`. Use `/fix-single` when you want to fix issues from a single-run report (`/review` or `/review-branch`) instead.

---

## Hard rules (MANDATORY)

- **Strict report selection order**:
  1) Resolve the latest merged report path using Step 0.1 selector.
  2) Print/store that exact path as `SELECTED_REPORT_PATH`.
  3) Only then read and parse the report JSON.
  - Do **not** read any `AUDIT_REPORT_*_MERGED.json` before Step 0.1 completes.
  - Do **not** use an IDE-open report file as implicit input.
- **Validate before fixing**: For every reported issue, first confirm it actually exists in the current codebase and is correctly interpreted.
- **No blind fixes**: If an issue is irrelevant, invalid, or misinterpreted, do **not** implement a fix. Instead, **push back** with clear, evidence-based reasons (including file/line references where applicable).
- **Sufficient, durable fixes**: Implement changes that address the issue **permanently and comprehensively** within this codebase's standards (not "minimal changes" by default).
- **Refactor decision gate**: If the best fix requires a larger refactor that might be better suited for a dedicated issue/SHI/branch:
  - Propose options: (A) short-term/minimal mitigation now, (B) comprehensive fix now, (C) raise/follow-up SHI for later.
  - Explain trade-offs (risk, scope, time, testing impact).
  - Then **ask the user which option they want** before proceeding with a significant refactor.
- **No surprise commits**: Do not create commits or push unless the user explicitly asks.
- **Do not create report files**: Do **not** write `.md` or `.json` report artifacts. The fix report must be provided **in chat output only**.
- **Plain English output**: All outputs must be in plain English and structured so a reader can quickly understand what was wrong, what you verified, what you changed & why, and what remains.

---

## Step 0: Load the audit issues (MANDATORY)

1. Locate the latest **merged** audit report JSON and treat it as the only valid input (`SELECTED_REPORT_PATH`):

```bash
python - <<'PY'
from pathlib import Path
import sys

root = Path("Output/CodeReview")
# Preferred location (current): inside run directories.
# IMPORTANT: choose "latest" by RUN_ID / filename (lexicographic timestamp), not mtime.
# mtime can be unreliable (e.g. file touched/copied after the newest run completes).
run_candidates = sorted(root.glob("RUN_*/AUDIT_REPORT_*_MERGED.json"))
if run_candidates:
    # Sort by parent RUN_<RUN_ID> folder name first, then filename.
    # RUN_ID starts with UTC timestamp: YYYYMMDDTHHMMSSZ-<suffix>, so lexicographic ordering works.
    run_candidates.sort(key=lambda p: (p.parent.name, p.name), reverse=True)
    print(run_candidates[0].as_posix())
    raise SystemExit(0)

raise SystemExit("No merged audit report found under Output/CodeReview/ (run /review-parallel or /review-branch-parallel first, or use /fix-single for single-run reports)")
PY
```

2. Capture the printed path as `SELECTED_REPORT_PATH`. If you are about to read a different merged report path, stop and restart from Step 0.1.
3. Open and parse `SELECTED_REPORT_PATH` (and no other merged report as the primary input).
4. Extract:
   - The audit `verdict`
   - The full `findings[]` list (use `id` as the canonical issue number)
   - Each finding's `priority`, `title`, `categories`, `locations`, `problem`, `fix` (guidance)

5. **Linear requirements context (MANDATORY — do not silently skip)**:

   a) **Extract SHI keys** to verify against (before validating any findings):
   - Scan for keys matching `SHI-\\d+` in:
     - The user request (if present)
     - The parsed report (including `scope.branch` and all `findings[*].title/problem/fix`)
   - Normalize common variants to `SHI-###`:
     - `shi-172/...` (branch names) → `SHI-172`
     - `shi_172`, `shi172` → `SHI-172`
   - If the report’s `contextDocumentsReviewed` references a run `shared_context.md`, read it and scan it too:
     - `Output/CodeReview/RUN_<RUN_ID>/shared_context.md`

   b) **Fetch Linear for every SHI key found**:
   - Retrieve the issue **body AND comments** using Linear tools (if direct lookup fails, search issues by query).
   - Treat comments as authoritative clarifications.
   - If you cannot retrieve Linear (auth/tool failure), you MUST explicitly say so in the fix report. If SHI keys were detected, do **not** proceed with requirements-sensitive validation silently; either stop or proceed only with a clearly marked “requirements not verified” caveat (user decision if needed).

   c) **Always report what you did**:
   - In your fix report, explicitly state one of:
     - “Linear context fetched for: SHI-###, …” (and whether comments were reviewed), or
     - “No SHI keys detected; Linear fetch skipped.”

If `findings` is empty, stop and report: "No issues to fix (findings[] is empty)."

---

## Step 1: Validate each issue exists (MANDATORY)

For each finding (Issue N):

- Locate the referenced code (`locations[]`) and confirm the problem is present in the current codebase.
- If the location is vague/incorrect, use the title/problem to find the correct location(s) and document what you found.
- Decide one of these outcomes:
  - **CONFIRMED**: the issue is real and should be fixed
  - **NOT AN ISSUE**: the report is incorrect / misinterpreted / not applicable (push back with evidence)
  - **ALREADY FIXED**: the issue is no longer present (explain why)
  - **NEEDS CLARIFICATION**: the report is ambiguous or depends on requirements you cannot infer (state the minimal questions needed)

Do not proceed to implementation until you have completed validation for all issues, unless an issue is clearly independent and safe to fix immediately.

---

## Step 2: Implement fixes for CONFIRMED issues

For each **CONFIRMED** issue:

- Implement the fix in the most direct, maintainable way that meets repo standards.
- Prefer durable fixes over band-aids (unless you deliberately choose a short-term mitigation via the refactor decision gate).
- Add or update tests where appropriate to prevent regression.
- Ensure docs/config/tests remain aligned where the change affects externally observable behavior.

If a fix would require a broad refactor, use the **Refactor decision gate** above before proceeding.

---

## Step 3: Verify (MANDATORY)

Run the smallest set of commands that provides confidence (prefer targeted verification).

Examples (pick what fits the repo):

```bash
dotnet build -c Release
dotnet test -v minimal
```

If you cannot run verification, state exactly what should be run by the author and why.

---

## Step 4: Fix report (CHAT OUTPUT ONLY, MANDATORY)

Provide a report in chat that is **polished, skimmable, and plain-English**, but focused on validation + fixes.

Use this exact format:

---

### FIX SUMMARY

- **Input**: `<exact SELECTED_REPORT_PATH from Step 0.1>`
- **Issues total**: X
- **Confirmed & fixed**: A
- **Not issues (pushed back)**: B
- **Already fixed**: C
- **Needs clarification**: D

Input integrity check:
- Confirm the report analyzed for all validations/fixes is the same as `**Input**`.
- If a different report path was used at any point, declare it and restart using the correct `SELECTED_REPORT_PATH`.

---

### VERIFICATION

- Build/tests run: ✅/❌ (list commands)
- Notes: [brief]

---

### RESULTS BY ISSUE

For each issue in numeric order (Issue 1, Issue 2, ...):

## Issue N: <title>

| | |
|---|---|
| 🏷️ **Priority** | P0/P1/P2/P3 |
| 🏷️ **Categories** | Primary; Secondary (optional) |
| 🔎 **Validation** | CONFIRMED / NOT AN ISSUE / ALREADY FIXED / NEEDS CLARIFICATION |
| 📍 **Location(s)** | `path:line` (or best-known location(s)) |
| 🧾 **Evidence** | Short, concrete evidence of what you found |
| ✅ **Fix applied** | What changed (or "N/A") |
| 🧪 **Tests/verification** | What you ran/added, or "Not run (reason)" |

If you pushed back (NOT AN ISSUE), include a brief "Why the report is wrong" paragraph with evidence.
If NEEDS CLARIFICATION, include the minimal questions needed to proceed safely.

---

### REMAINING RISKS / FOLLOW-UPS (if any)

- List any residual risks, trade-offs, or deferred work (keep it short).

```

