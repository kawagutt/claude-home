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

* Start from a clean working tree on a new branch. Verify there are no uncommitted changes, then create and switch to a new branch before implementing. Do all implementation there.
* If the working tree is dirty, the directory is not a git repository, or a branch cannot be created, stop and ask the user how to proceed instead of implementing in place. Do not stash, discard, or commit the user's existing changes on your own.
* Do not touch untracked files not created in this session unless the user explicitly asks.
* Use a fresh Implementer subagent for implementation when available.
* Reviewer agents must be read-only and must not fix issues.
* Fixes must be made by an Implementer, not a Reviewer.
* Prefer simple, scoped changes.
* Preserve fail-fast behavior.
* Do not add fallback behavior, hidden normalization, or silent auto-correction unless requested.
* Run meaningful verification before declaring completion.
* Save a record of the run (reviews, triage, verification) under `~/.claude/plans/...`, outside the target repository. Record only what carries signal; skip meaningless or empty records.

# Process

1. Validate input and prepare a clean branch.

   * If the plan is missing or too vague, stop and ask the user to run `/plan` or provide a clearer plan.
   * Identify likely files and areas the plan will touch.
   * Confirm the git working tree is clean (`git status`). If there are uncommitted changes, or this is not a git repository, stop and ask the user how to proceed. Do not stash, discard, or commit existing changes on your own.
   * Create and switch to a new branch named descriptively from the task, and do all implementation there.
   * If a branch cannot be created, or the user prefers to work in place, follow the user's direction.

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

7. Save the run record.

   * Save the record under the same session directory as the plan: `~/.claude/plans/<project>/<yyyymmdd>_<slug>/`. If the plan came from a saved `plan_*.md`, reuse that directory; otherwise derive and create it as in the `/plan` save step. This is outside the target repository, so it never dirties or gets committed to the project.
   * Choose the smallest `N` starting at 0 for which `implementation_v<N>.md` does not exist; never overwrite an existing version. Write the record to `implementation_v<N>.md`, then copy it to `implementation_latest.md`.
   * Include only what carries signal: a brief summary of what changed, meaningful review and test-review findings and how each was triaged (fixed, or rejected with the reason), any fix/review rounds, and the verification commands with their results. Omit sections that add no information.
   * Skip the record entirely when there is nothing meaningful to capture, such as a trivial change with no findings and an obvious passing check. Do not create empty or boilerplate records.

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

## Saved record

State the path of the `implementation_v<N>.md` written, and that `implementation_latest.md` mirrors it. If no record was warranted, say so in one line.
