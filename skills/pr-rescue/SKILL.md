---
name: PR Rescue
description: Use when reviewing, fixing, or verifying a pull request. Focus on correctness regressions, missing tests, permission mistakes, unsafe behavior, and minimal fixes.
allowed-tools: memory_search
tags:
  - pull-request
  - code-review
  - bug-fix
  - testing
  - self-improvement
---

# PR Rescue Skill

Use this skill when an agent must rescue a pull request.

The goal is one high-confidence finding with a minimal fix and a named test. Do not produce long reviews. Do not speculate.

## Process

1. List every changed file and identify which public APIs, workflows, or storage layers each one touches.
2. Call `memory_search` with the primary changed symbol before making any claim about its behavior.
3. Scan for correctness regressions first — inputs that used to work and now fail.
4. Check in order: empty/null inputs, permission and tenant isolation, auth boundaries, destructive operations, partial failure consistency.
5. For every storage write (graph, vector, relational, cache, file): verify the write is safe when the input collection is empty.
6. Check that at least one test targets the changed behavior directly — not just the happy path.
7. Separate confirmed findings (you can point to the line) from guesses (you cannot). Drop guesses.
8. Pick the single highest-confidence finding. One finding done well beats five speculative ones.
9. Propose the smallest patch: ideally one guard clause, one check, or one early return.
10. Name the exact test case that must pass after the fix — file, function, and assertion.

## Review Checklist

For every changed file, ask:

- What input used to work that might now fail?
- What happens when the input collection is empty, None, or missing?
- Does this path write to graph, vector, relational, cache, or file storage?
- Can a partial write or partial failure leave inconsistent state?
- Are user, tenant, dataset, and permission checks still in place after this change?
- Does the PR's test prove the changed behavior, or only the happy path?
- Is there a destructive operation (delete, overwrite, truncate) with no guard?
- Does any decoded token or credential skip signature verification?

## Output Format

**Always include all six fields for every finding.**

```
Severity: critical | high | medium | low
Location: <file path>:<line> — <symbol or function name>
Problem: <one sentence — what breaks and for whom>
Evidence: <point to the specific line or condition that causes it>
Fix: <smallest practical change — prefer a guard clause or early return>
Test: <exact test to add — file, function name, and what it asserts>
```

If no issues are found, say so directly. Then list remaining test gaps and residual risk explicitly.

Do not pad the output. Do not add sections not listed above.

## Rules Added From Prior Run Feedback

*(This section grows after each run. Each rule is one sentence.)*

- When a PR touches graph writes, verify that passing an empty edge list does not raise an exception or silently corrupt state.
- When reviewing auth or token handling, confirm that signature verification is called before any claim in the decoded payload is trusted.
- When a PR changes permission checks, explicitly name the user boundary, tenant boundary, dataset boundary, and role being checked.

## Self-Improvement Instructions

After each run, read the `SkillRunEntry` score and feedback field.

If `success_score < 0.7`, add exactly one rule to the **Rules Added From Prior Run Feedback** section above.

The rule must:
- Be one sentence.
- Name the specific condition that caused the failure.
- Tell the next agent what to check that this agent missed.

Do not add vague rules. Do not rewrite the whole skill. Change only what the feedback identifies.
