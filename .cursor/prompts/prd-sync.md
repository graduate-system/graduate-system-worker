# PRD Sync Issue Investigation

I've been assigned a task to resolve all issues under the Linear project named **PRDSync1**. You will be the agent assisting me to resolve these issues.

I now want us to work on **SHI-XX**.

---

## Your Task

### Step 1: Investigate

First, answer these questions:

1. **Does this issue exist?** — Verify whether the issue is real and valid in this codebase. If not, explain why.
2. **PRD vs. Reality** — What does the PRD say should happen? What is the code actually doing? Where's the gap?

Then, explicitly enumerate the gaps you found as a numbered list so they can be referenced later in the write-up:

## Issues (numbered)

1. **Issue 1** — <short title>
2. **Issue 2** — <short title>
   - Only include additional issues if they are truly distinct; don't over-split.

### Step 2: Produce an Explainer

If the issue is valid, produce an **explainer** that helps someone — even someone completely new to the codebase — quickly understand and act on this issue.

Your explainer must enable a reader to quickly understand:

1. **How the system works** — Enough context about the relevant part of the system to understand the issue
2. **What the issue is** — A clear explanation of what's going wrong or what's missing
3. **Why it's actually an issue** — The real-world impact, who's affected, and which requirements are violated
4. **The options for resolving it** — At least two approaches, with pros and cons of each
5. **Your recommended option** — Which approach you'd choose and why

---

## Format Guidelines

You have flexibility in how you structure the explainer — adapt it to what makes sense for the specific issue. Some issues need state diagrams; others need code snippets; others need before/after comparisons.

**What matters most:**
- Someone new to the codebase can understand it in one read
- Use plain English, not jargon
- Show concrete examples (actual file paths, status values, code paths)
- Include a summary table for quick reference
- List the key files involved so the reader knows where to look
- Keep issue numbering consistent (e.g., if you refer to "Issue 2" in one section, it must refer to the same issue everywhere)

**Useful techniques (use when appropriate):**
- State diagrams for workflow issues
- "Happy path vs failure scenario" comparisons for bugs
- Tables for comparing options or summarizing states
- Code snippets when the root cause is in specific logic
- Mermaid diagrams for complex flows

---

## Checklist Before Finishing

- [ ] Could someone new to the codebase understand this in one read?
- [ ] Is the root cause clearly identified?
- [ ] Are at least two resolution options presented with pros/cons?
- [ ] Is there a clear recommendation?
- [ ] Are the key files listed so the reader knows where to look?
- [ ] If database tables are involved, have you verified the actual schema?

---

## Example

Here's an example of a well-structured explainer. You don't need to follow this exact structure — adapt based on what best explains your specific issue.

---

# SHI-82: Invoice Resumption Bug — Explainer

This document explains the SHI-82 issue for someone new to the codebase.

---

## What does this system do?

The **billing generator** is an Azure Function that automatically generates invoice PDFs and records delivery intent via a database outbox (A2: Generate → Outbox → Send). It runs on a schedule (daily) or can be triggered manually via HTTP.

Invoice email delivery is owned by a separate **sender worker** that claims pending outbox rows and sends via Microsoft Graph.

For each customer, the system:

1. Figures out which invoices are due
2. Calculates line items and totals
3. Generates a PDF
4. Uploads the PDF to blob storage
5. Creates/updates a delivery outbox row (`pending` when deliverable, otherwise `cancelled` with `last_error`)
6. (Sender worker) Sends the invoice email and records the invoice as "sent"

---

## How does an invoice move through the system?

An invoice goes through these **statuses**:

```
draft → generated → sent
          ↘
           failed_generate / failed_send (if something goes wrong)
```

| Status | Meaning |
|--------|---------|
| `draft` | Slot reserved in DB; processing has started |
| `generated` | PDF created/uploaded and delivery intent is recorded (outbox row exists) |
| `sent` | Email successfully delivered (sender worker) |
| `failed_generate` | Something went wrong during generation (DB/PDF/blob phase) |
| `failed_send` | Something went wrong during delivery (sender/outbox phase) |

The system also assigns an **invoice number** (like `INV-2025-000123`) partway through processing—after calculating totals but before generating the PDF.

---

## What is "idempotency" and why does it matter?

**Idempotency** means: if you run the same operation twice, you get the same result (no duplicates, no double-billing).

For invoicing, this is critical because:

- We **never** want to send two invoices for the same billing period
- We **never** want to email the same invoice twice
- Multiple worker instances might try to process the same invoice simultaneously

The system enforces this by:

1. Using a database constraint to prevent duplicate invoice records
2. Checking if an invoice already exists before creating a new one

---

## What's the bug?

The system has a **resumption problem**. Here's the scenario:

### The happy path (no crash)

```
1. Reserve invoice slot (status = draft)
2. Calculate totals
3. Assign invoice number (e.g., INV-2025-000123)  ← number saved to DB
4. Generate PDF
5. Upload to blob storage
6. Mark as "generated"
7. Send email
8. Mark as "sent"  ✓ Done!
```

### The crash scenario

```
1. Reserve invoice slot (status = draft)
2. Calculate totals
3. Assign invoice number (e.g., INV-2025-000123)  ← number saved to DB
4. Generate PDF
5. Upload to blob storage
   💥 CRASH (e.g., timeout, out of memory, deployment)
```

Now the invoice is stuck: it has a number, maybe a PDF, but was never sent.

### What happens on the next run?

The system checks: "Does an invoice already exist for this billing period?"

- Answer: **Yes** (status = `draft` or `generated`, with invoice number `INV-2025-000123`)

**Expected behavior (per the PRD):** Resume processing—use the existing invoice number, skip steps already done, finish sending.

**Actual behavior:** The system sees that an invoice number already exists and **skips** the invoice entirely, logging "already in progress."

### Why?

The code has this logic:

```
If invoice exists AND has no invoice number yet → resume it
If invoice exists AND already has an invoice number → skip it
```

The second condition is the bug. The system was designed to only resume invoices that crashed *before* getting a number, not after.

---
 
## Why is this a problem?

1. **Invoices get stuck**: They sit in `draft`/`generated`/`failed_generate`/`failed_send` status forever
2. **Manual intervention required**: An operator must manually force-regenerate or fix each stuck invoice
3. **PRD violation**: The requirements document (NFR-5.2) says the system "SHALL be able to idempotently pick up and complete the work" after a mid-way failure—but that's not happening

---

## The two options to fix this

### Option A: Make resumption work properly

Change the code to:

- Resume processing even when an invoice number exists
- Reuse the existing invoice number (don't generate a new one)
- Make delivery intent/outbox upsert idempotent (so retries are safe) and ensure email sending is idempotent in the sender loop (so we don't accidentally send twice)

**Pros:** Matches what the PRD requires; fewer stuck invoices; less manual work.

**Cons:** Requires careful handling of "did we already send the email?" to avoid duplicates. May need a small database change to track email send status reliably.

### Option B: Change the requirements to match the code

Update the PRD and test plan to say:

> "Resumption only works if the crash happens before the invoice number is assigned. After that, operator intervention is required."

**Pros:** No code changes; quick to close the issue.

**Cons:** Doesn't fix the underlying problem; more manual work for ops; invoices can still get stuck.

---

## Summary

| | |
|---|---|
| **Issue** | Invoices that crash mid-processing (after getting a number) are skipped instead of resumed |
| **Impact** | Stuck invoices requiring manual intervention |
| **Root cause** | Code only resumes invoices without an invoice number |
| **PRD says** | System SHALL resume mid-way failures (NFR-5.2) |
| **Code does** | Only resumes if no invoice number assigned yet |

---

## Key files involved

| File | Role |
|------|------|
| `Services/IdempotenceService.cs` | Checks for existing invoices, returns status |
| `Services/InvoiceGenerationService.cs` | Orchestrates invoice generation and outbox enqueueing; contains the resumption logic |
| `Agile/Docs/Requirements/PRD-agile-billing-automation.md` | NFR-5.2 defines the resumption requirement |
| `Docs/Plans/test-plan.md` | E2E-IDEM-003 tests crash recovery (but only for pre-number crashes) |

---

## Related links

- [SHI-82 in Linear](https://linear.app/shipht/issue/SHI-82/medium-nfr-52-idempotent-resumption-not-fully-implemented-skips-when)

