---
name: review
version: 1.0.0
description: |
  Pre-landing PR review. Analyzes diff against main for SQL safety, LLM trust
  boundary violations, conditional side effects, and other structural issues.
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - Grep
  - Glob
  - AskUserQuestion
---

# Pre-Landing PR Review

You are running the `/review` workflow. Analyze the current branch's diff against main for structural issues that tests don't catch.

---

## Step 1: Check branch

1. Run `git branch --show-current` to get the current branch.
2. If on `main`, output: **"Nothing to review — you're on main or have no changes against main."** and stop.
3. Run `git fetch origin main --quiet && git diff origin/main --stat` to check if there's a diff. If no diff, output the same message and stop.

---

## Step 2: Read the checklist

Use the embedded checklist in the `## Embedded Checklist` section below.

**Do not skip the checklist.** It is part of this skill prompt so the review stays self-contained inside a Cursor plugin install.

---

## Step 3: Get the diff

Fetch the latest main to avoid false positives from a stale local main:

```bash
git fetch origin main --quiet
```

Run `git diff origin/main` to get the full diff. This includes both committed and uncommitted changes against the latest main.

---

## Step 4: Two-pass review

Apply the checklist against the diff in two passes:

1. **Pass 1 (CRITICAL):** SQL & Data Safety, LLM Output Trust Boundary
2. **Pass 2 (INFORMATIONAL):** Conditional Side Effects, Magic Numbers & String Coupling, Dead Code & Consistency, LLM Prompt Issues, Test Gaps, View/Frontend

Follow the output format specified in the checklist. Respect the suppressions — do NOT flag items listed in the "DO NOT flag" section.

---

## Step 5: Output findings

**Always output ALL findings** — both critical and informational. The user must see every issue.

- If CRITICAL issues found: output all findings, then for EACH critical issue use a separate AskUserQuestion with the problem, your recommended fix, and options (A: Fix it now, B: Acknowledge, C: False positive — skip).
  After all critical questions are answered, output a summary of what the user chose for each issue. If the user chose A (fix) on any issue, apply the recommended fixes. If only B/C were chosen, no action needed.
- If only non-critical issues found: output findings. No further action needed.
- If no issues found: output `Pre-Landing Review: No issues found.`

---

## Important Rules

- **Read the FULL diff before commenting.** Do not flag issues already addressed in the diff.
- **Read-only by default.** Only modify files if the user explicitly chooses "Fix it now" on a critical issue. Never commit, push, or create PRs.
- **Be terse.** One line problem, one line fix. No preamble.
- **Only flag real problems.** Skip anything that's fine.

---

## Embedded Checklist

### Instructions

Review the `git diff origin/main` output for the issues listed below. Be specific — cite `file:line` and suggest fixes. Skip anything that's fine. Only flag real problems.

**Two-pass review:**
- **Pass 1 (CRITICAL):** Run SQL & Data Safety and LLM Output Trust Boundary first. These can block `/ship`.
- **Pass 2 (INFORMATIONAL):** Run all remaining categories. These are included in the PR body but do not block.

**Output format:**

```text
Pre-Landing Review: N issues (X critical, Y informational)

**CRITICAL** (blocking /ship):
- [file:line] Problem description
  Fix: suggested fix

**Issues** (non-blocking):
- [file:line] Problem description
  Fix: suggested fix
```

If no issues found: `Pre-Landing Review: No issues found.`

Be terse. For each issue: one line describing the problem, one line with the fix. No preamble, no summaries, no "looks good overall."

### Review Categories

#### Pass 1 — CRITICAL

##### SQL & Data Safety
- String interpolation in SQL (even if values are `.to_i`/`.to_f` — use `sanitize_sql_array` or Arel)
- TOCTOU races: check-then-set patterns that should be atomic `WHERE` + `update_all`
- `update_column`/`update_columns` bypassing validations on fields that have or should have constraints
- N+1 queries: `.includes()` missing for associations used in loops/views (especially avatar, attachments)

##### Race Conditions & Concurrency
- Read-check-write without uniqueness constraint or `rescue RecordNotUnique; retry` (e.g., `where(hash:).first` then `save!` without handling concurrent insert)
- `find_or_create_by` on columns without unique DB index — concurrent calls can create duplicates
- Status transitions that don't use atomic `WHERE old_status = ? UPDATE SET new_status` — concurrent updates can skip or double-apply transitions
- `html_safe` on user-controlled data (XSS) — check any `.html_safe`, `raw()`, or string interpolation into `html_safe` output

##### LLM Output Trust Boundary
- LLM-generated values (emails, URLs, names) written to DB or passed to mailers without format validation. Add lightweight guards (`EMAIL_REGEXP`, `URI.parse`, `.strip`) before persisting.
- Structured tool output (arrays, hashes) accepted without type/shape checks before database writes.

#### Pass 2 — INFORMATIONAL

##### Conditional Side Effects
- Code paths that branch on a condition but forget to apply a side effect on one branch.
- Log messages that claim an action happened but the action was conditionally skipped.

##### Magic Numbers & String Coupling
- Bare numeric literals used in multiple files — should be named constants documented together
- Error message strings used as query filters elsewhere

##### Dead Code & Consistency
- Variables assigned but never read
- Version mismatch between PR title and VERSION/CHANGELOG files
- CHANGELOG entries that describe changes inaccurately
- Comments/docstrings that describe old behavior after the code changed

##### LLM Prompt Issues
- 0-indexed lists in prompts
- Prompt text listing tools/capabilities that do not match what is actually wired up
- Word/token limits stated in multiple places that could drift

##### Test Gaps
- Negative-path tests that assert type/status but not the side effects
- Assertions on string content without checking format
- `.expects(:something).never` missing when a code path should explicitly NOT call an external service
- Security enforcement features without integration tests verifying the enforcement path works end-to-end

##### Crypto & Entropy
- Truncation of data instead of hashing
- `rand()` / `Random.rand` for security-sensitive values
- Non-constant-time comparisons (`==`) on secrets or tokens

##### Time Window Safety
- Date-key lookups that assume "today" covers 24h
- Mismatched time windows between related features

##### Type Coercion at Boundaries
- Values crossing Ruby→JSON→JS boundaries where type could change
- Hash/digest inputs that do not normalize types before serialization

##### View/Frontend
- Inline `<style>` blocks in partials
- O(n*m) lookups in views
- Ruby-side `.select{}` filtering on DB results that should be a `WHERE` clause

### Gate Classification

```text
CRITICAL (blocks /ship):          INFORMATIONAL (in PR body):
├─ SQL & Data Safety              ├─ Conditional Side Effects
├─ Race Conditions & Concurrency  ├─ Magic Numbers & String Coupling
└─ LLM Output Trust Boundary      ├─ Dead Code & Consistency
                                   ├─ LLM Prompt Issues
                                   ├─ Test Gaps
                                   ├─ Crypto & Entropy
                                   ├─ Time Window Safety
                                   ├─ Type Coercion at Boundaries
                                   └─ View/Frontend
```

### Suppressions — DO NOT flag these

- Redundancy that is harmless and aids readability
- Comment-only requests explaining threshold choices
- Assertions that are already sufficiently tight for the behavior
- Consistency-only changes with no behavioral impact
- Regex edge cases that cannot happen in practice because inputs are constrained
- Tests that exercise multiple guards simultaneously
- Eval threshold tuning changes
- Harmless no-ops
- Anything already addressed in the diff being reviewed
