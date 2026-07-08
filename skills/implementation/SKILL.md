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
* After each subagent returns, print a short progress note to the user before continuing: a first line naming the subagent that finished, then a few bullet points summarizing its result (for an implementer, what it changed; for a reviewer, its verdict and top findings; for the finding-verifier, confirmed/adjusted/rejected counts). Keep it to a handful of lines; do not dump the subagent's full output.
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

   Then launch six independent reviewers covering two model perspectives, in parallel. Five specialized reviewers run on gpt-5.5, each on one concern so every perspective gets a focused pass; one holistic reviewer runs on claude-opus-4-8 as the second-model second opinion.

   gpt-5.5 specialized reviewers — each has `model: opus` frontmatter that resolves to gpt-5.5, so spawn them normally with no model override:

   * Spec Reviewer: use the `spec-reviewer` subagent and the `implementation-spec-review` skill — does the code do what the plan and request asked; correctness and behavioral coverage.
   * Architecture Reviewer: use the `architecture-reviewer` subagent and the `implementation-architecture-review` skill — structural soundness, boundaries, and fit with the existing design.
   * Refactor Reviewer: use the `refactor-reviewer` subagent and the `implementation-refactor-review` skill — naming consistency, duplication, unnecessary fallback, and over-abstraction, behavior-preserving only.
   * Environment Reviewer: use the `environment-reviewer` subagent and the `implementation-environment-review` skill — dependencies, configuration, compatibility, and permissions.
   * Test Reviewer: use the `test-reviewer` subagent and the `implementation-test-review` skill — coverage, edge cases, and verification adequacy.

   claude-opus-4-8 holistic reviewer:

   * Holistic Reviewer: use the `impl-holistic-reviewer` subagent for a single combined pass over all of the above concerns. Spawn it with the model set to the `sonnet` alias (remapped to claude-opus-4-8 via `ANTHROPIC_DEFAULT_SONNET_MODEL`). This is the independent second-model second opinion.

   Reviewers are read-only. They report issues only.

5. Verify findings before triage.

   Combine the six reviewers' findings and deduplicate. For each meaningful finding, spawn a `finding-verifier` subagent to independently confirm it, deepen it, decide whether it is actually a false positive, and sweep the codebase for similar occurrences. The verifier is read-only and reports a per-finding verdict.

   * Verify each finding with the *opposite* model from the reviewer that raised it, so no finding is checked by the model that produced it:
     * findings from the five gpt-5.5 specialized reviewers → spawn the verifier with the `sonnet` alias (claude-opus-4-8).
     * findings from the claude-opus-4-8 holistic reviewer → spawn the verifier with `model: opus` (gpt-5.5).
   * Choose fan-out granularity by weight: one verifier per blocking or significant finding for depth; batch closely related low-severity findings into a single verifier call. Skip verifying findings that are trivially out of scope.
   * Carry forward each verdict: drop false positives (record why), keep confirmed findings with their adjusted severity, and promote any similar occurrences the verifier surfaced to new findings of the same class.

6. Triage verified findings.

   * Work from the verified set. Note where the gpt-5.5 specialized reviewers and the claude-opus-4-8 holistic reviewer agree or disagree, and weigh disagreements on their merits rather than by majority.
   * Ignore findings that are clearly out of scope or that verification refuted, and explain why.
   * Send valid findings back to an Implementer for fixes.
   * Re-review significant fixes when needed.
   * Limit fix/review loops to two unless the user asks for more.

7. Final verification.

   * Run relevant tests, type checks, linters, or direct behavior checks.
   * For nontrivial product changes, verify the affected flow end-to-end when practical.
   * Report exact failures if verification fails.

8. Save the run record.

   * Save the record under the same session directory as the plan: `~/.claude/plans/<project>/<yyyymmdd>_<slug>/`. If the plan came from a saved `plan_*.md`, reuse that directory; otherwise derive and create it as in the `/plan` save step. This is outside the target repository, so it never dirties or gets committed to the project.
   * Choose the smallest `N` starting at 0 for which `implementation_v<N>.md` does not exist; never overwrite an existing version. Write the record to `implementation_v<N>.md`, then copy it to `implementation_latest.md`.
   * Include only what carries signal: a brief summary of what changed, meaningful findings from the six reviews and the finding-verification verdicts (confirmed, adjusted, or rejected as false positive) and how each was triaged (fixed, or rejected with the reason), any fix/review rounds, and the verification commands with their results. Omit sections that add no information.
   * Skip the record entirely when there is nothing meaningful to capture, such as a trivial change with no findings and an obvious passing check. Do not create empty or boilerplate records.

# Final output

Return:

## Completed

Summarize what changed.

## Reviews

Summarize the outcomes across the five gpt-5.5 specialized reviews (spec, architecture, refactor, environment, test) and the claude-opus-4-8 holistic review, then the finding-verification results: which findings were confirmed, adjusted, or rejected as false positives, and any similar occurrences the verifier surfaced. Call out any material disagreement between the two model perspectives.

## Verification

List commands or checks run and their results.

## Notes

Mention remaining risks, assumptions, or follow-up work only if relevant.

## Saved record

State the path of the `implementation_v<N>.md` written, and that `implementation_latest.md` mirrors it. If no record was warranted, say so in one line.
