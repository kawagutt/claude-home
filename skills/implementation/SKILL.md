---
name: implementation
description: Orchestrate implementation from an approved plan using a fresh implementer, risk-based review, fixes, and final verification.
argument-hint: "<plan-file or approved plan>"
disable-model-invocation: true
---

# Goal

Implement an approved plan with separation between implementation and review.

## Input

$ARGUMENTS

# Role

Act as the orchestrator. Coordinate implementation, independent review, fixes, and final verification. Keep responsibility boundaries clear.

# Rules

* Run immediate preflight before any implementation work.
* Do not stash, discard, commit, or otherwise clean up the user's existing changes on your own.
* Do not create or switch branches without user approval.
* After preflight passes, implement and fix without asking before each file edit or write. Only stop for approval on branch creation, dirty working tree handling, or destructive git/filesystem operations. Preflight must pass before any `Edit` or `Write`.
* Do not touch untracked files not created in this session unless the user explicitly asks.
* Use a fresh Implementer subagent for implementation when available.
* Reviewer agents must be read-only and must not fix issues.
* Fixes must be made by an Implementer, not a Reviewer.
* After each subagent returns, print a short progress note to the user before continuing: a first line naming the subagent that finished, then a few bullet points summarizing its result (for an implementer, what it changed; for a reviewer, its verdict and top findings; for the finding-verifier, confirmed/adjusted/rejected counts). Keep it to a handful of lines; do not dump the subagent's full output.
* Prefer simple, scoped changes.
* Preserve fail-fast behavior.
* Do not add fallback behavior, hidden normalization, or silent auto-correction unless requested.
* Run meaningful verification before declaring completion.
* Save a record of the run under `<project>/.claude/workflows/...` in the same session directory as the plan. Record only what carries signal; skip meaningless or empty records.
* Choose the lightest review profile that fits the change. Do not run every reviewer on every task.
* When using a second-model reviewer or verifier, do not assume the requested per-invocation model was honored. If `CLAUDE_CODE_SUBAGENT_MODEL` is set, it may override per-invocation and agent-frontmatter model choices. Before high-risk second-model review, check whether that variable forces all subagents to one model. If second-model independence cannot be guaranteed, warn the user instead of claiming a second-model review.

# Process

1. Immediate preflight.

   Do this before launching subagents or making edits.

   **Resolve target project**

   * If the argument is a file path, note the plan path.
   * If the argument is inline text, use it directly. Use the current git repository root as the target project root. If the current directory is not inside a git repository, use the current working directory and follow the non-git confirmation rule below.
   * If the plan is missing or too vague, stop and ask the user to run `/plan` or provide a clearer plan.
   * If the argument is a plan file path under `<project>/.claude/workflows/...`, derive `<project>` from that path. The project root is the path segment immediately before `.claude/workflows/`.
   * Before running `git status` or making edits, verify the current working directory is inside the target project root. If it is not, stop and ask the user to switch to the correct project directory.

   **Dirty check (first)**

   * Run `git status` from the target project root immediately (for example, `git -C <project> status`).
   * Ignored paths—including `.claude/workflows/` when excluded via `.gitignore`, `.git/info/exclude`, or global exclude—do not count as dirty.

   If the working tree is dirty outside ignored paths:

   * stop immediately; do not implement
   * do not stash, discard, commit, or delete anything on your own
   * list the dirty files briefly
   * ask the user how to proceed
   * wait for explicit cleanup or handling instructions

   **Branch decision (after clean check)**

   After the working tree is confirmed clean, read the plan only enough to classify task size and risk. Then recommend whether to create a new branch. Do not do deep implementation analysis or launch subagents before the branch decision is resolved.

   * say the working tree is clean
   * recommend whether to create a new branch based on task risk:
     * recommend a new branch for non-trivial, multi-file, or behavior-changing work
     * say the current branch is fine for very small localized fixes
   * ask for approval before creating or switching branches
   * do not create or switch branches on your own

   Example when recommending a branch:

   ```text
   The working tree is clean. I recommend creating a new branch because this is a multi-file behavior change. Should I create one now, or continue on the current branch?
   ```

   If this is not a git repository, continue only after the user confirms how to proceed.

2. Choose checkpoint size.

   * Small task: implement all planned tasks, then review once.
   * Larger task: implement natural checkpoints and review each checkpoint.
   * Do not default to per-task multiple reviewers when that would be unnecessarily heavy.

3. Launch a fresh Implementer subagent.

   Use the `implementer` custom subagent when available. Provide:

   * the approved plan
   * current repository constraints
   * instruction to use `implementation-execution` and `implementation-tdd` practices
   * scope boundaries

4. Choose a review profile.

   Classify the change and run only the reviewers that fit. Default to the lightest profile that matches.

   | Profile | When to use | Reviewers |
   | --- | --- | --- |
   | Small | localized fix, few files, no contract change | `spec-reviewer`, `test-reviewer` |
   | Medium | multi-file or moderate behavior change | above + `refactor-reviewer` |
   | Structural / risky | layering, public contract, schema, or design changes | above + `architecture-reviewer` |
   | Config / dependency / runtime | deps, CI, scripts, env, permissions | `spec-reviewer`, `environment-reviewer`, `test-reviewer` |
   | High-risk / confusing | security-sensitive, subtle correctness, or major cross-cutting risk | add `holistic-reviewer` on a second configured model |

   Config/runtime and structural profiles can combine when both apply.

   The orchestrator runs `git diff`, `git status`, and any needed verification commands. Reviewers use `Read`, `Grep`, and `Glob` only and cannot run shell commands themselves.

5. Run the selected reviewers in parallel when possible.

   Provide each reviewer with this **reviewer context package**:

   * approved plan or fix request
   * **unified diff with enough context** (required)
   * changed file list
   * git status summary
   * relevant test output or verification output
   * relevant project constraints from CLAUDE.md
   * any files or snippets the reviewer must inspect
   * explicit instruction to use its matching `implementation-*-review` skill

   Reviewers are read-only. They report issues only.

6. Verify findings only when needed.

   Combine reviewer findings and deduplicate.

   Run `finding-verifier` only for:

   * blocking findings
   * materially disputed findings between reviewers
   * high-risk runs where holistic review surfaced a significant concern worth confirming

   Do not verify every non-blocking finding by default.

   When verifying, spawn the verifier on a different configured model from the reviewer that raised the finding when second-model independence is available. If `CLAUDE_CODE_SUBAGENT_MODEL` prevents that, note the limitation in the review summary.

   Provide each verifier with:

   * the original finding
   * the unified diff
   * the list of changed files
   * relevant surrounding files or snippets
   * suspected patterns or search terms for sweeping similar occurrences

   Carry forward each verdict: drop false positives (record why), keep confirmed findings with their adjusted severity, and promote any similar occurrences the verifier surfaced to new findings of the same class.

7. Triage findings.

   * Work from the verified set when verification ran; otherwise use the raw reviewer findings.
   * Ignore findings that are clearly out of scope or that verification refuted, and explain why.
   * Send blocking and agreed meaningful findings back to an Implementer for fixes.
   * Re-review significant fixes when needed.
   * Limit fix/review loops to two unless the user asks for more.

8. Final verification.

   * Run relevant tests, type checks, linters, or direct behavior checks.
   * For nontrivial product changes, verify the affected flow end-to-end when practical.
   * Report exact failures if verification fails.

9. Save the run record.

   Store workflow artifacts inside the current project under:

   ```text
   <project>/.claude/workflows/<yyyymmdd>_<short-slug>/
   ```

   * Save the record under the same session directory as the plan. If the plan came from a saved `plan_*.md`, reuse that directory; otherwise derive and create it as in the `/plan` save step.
   * If `.claude/workflows/` is not ignored yet and this is the first workflow write in the session, follow the same `.git/info/exclude` recommendation as in `/shape` and `/plan` before writing (check with `git check-ignore -q .claude/workflows/ .claude/workflows/__check__`; add the exclude entry only if not already present). If the user declines exclude setup, ask whether to save anyway as untracked or stop without saving. Do not silently save an unignored workflow artifact.
   * Choose the smallest `N` starting at 0 for which `implementation_v<N>.md` does not exist; never overwrite an existing version. Write the record to `implementation_v<N>.md`, then copy it to `implementation_latest.md`.
   * Include only what carries signal: review profile used, what changed, meaningful findings and how they were triaged, any verification runs, and fix/review rounds. Omit sections that add no information.
   * Skip the record entirely when there is nothing meaningful to capture. Do not create empty or boilerplate records.

# Final output

Return:

## Completed

Summarize what changed.

## Reviews

State which review profile ran and summarize outcomes. If verification ran, note which findings were confirmed, adjusted, or rejected as false positives.

## Verification

List commands or checks run and their results.

## Notes

Mention remaining risks, assumptions, or follow-up work only if relevant.

## Saved record

State the path of the `implementation_v<N>.md` written, and that `implementation_latest.md` mirrors it. If no record was warranted, say so in one line.
