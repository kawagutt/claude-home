---
name: implementation
description: Orchestrate implementation from an approved plan using a fresh implementer, independent code and test review, fixes, and final verification.
argument-hint: "<approved implementation plan>"
disable-model-invocation: true
---

# Goal

Implement an approved plan with separation between implementation and review.

## Input

$ARGUMENTS

# Role

Act as the orchestrator. Coordinate implementation, independent review, fixes, and final verification. Keep responsibility boundaries clear.

# Rules

* Before starting, check whether relevant tracked files are already dirty. If they are, stop and ask before mixing edits.
* Do not touch untracked files not created in this session unless the user explicitly asks.
* Use a fresh Implementer subagent for implementation when available.
* Reviewer agents must be read-only and must not fix issues.
* Fixes must be made by an Implementer, not a Reviewer.
* Prefer simple, scoped changes.
* Preserve fail-fast behavior.
* Do not add fallback behavior, hidden normalization, or silent auto-correction unless requested.
* Run meaningful verification before declaring completion.

# Process

1. Validate input.

   * If the plan is missing or too vague, stop and ask the user to run `/plan` or provide a clearer plan.
   * Identify likely files and check their git status before editing.

2. Decide checkpoint size.

   * Small task: implement all planned tasks, then review once.
   * Larger task: implement natural checkpoints and review each checkpoint.
   * Do not default to per-task multiple reviewers when that would be unnecessarily heavy.

3. Launch a fresh Implementer subagent.

   Use the `implementer` custom subagent when available. Provide:

   * the approved plan
   * current repository constraints
   * instruction to use `implementation-execution` and `implementation-tdd` practices
   * scope boundaries

4. After implementation, run independent reviews in parallel when possible.

   Reviewers start in a fresh context. Provide each reviewer:

   * the approved plan or fix request
   * the diff or list of changed files
   * relevant repository constraints

   Then launch:

   * Code Reviewer: use `code-reviewer` custom subagent and the `implementation-code-review` skill.
   * Test Reviewer: use `test-reviewer` custom subagent and the `implementation-test-review` skill.

   Reviewers are read-only. They report issues only.

5. Triage findings.

   * Ignore findings that are clearly out of scope or incorrect, and explain why.
   * Send valid findings back to an Implementer for fixes.
   * Re-review significant fixes when needed.
   * Limit fix/review loops to two unless the user asks for more.

6. Final verification.

   * Run relevant tests, type checks, linters, or direct behavior checks.
   * For nontrivial product changes, verify the affected flow end-to-end when practical.
   * Report exact failures if verification fails.

# Final output

Return:

## Completed

Summarize what changed.

## Reviews

Summarize code review and test review outcomes, including any skipped or rejected findings.

## Verification

List commands or checks run and their results.

## Notes

Mention remaining risks, assumptions, or follow-up work only if relevant.
