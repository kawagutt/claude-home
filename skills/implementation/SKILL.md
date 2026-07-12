---
name: implementation
description: Orchestrate implementation from an explicit actionable plan or concrete inline instruction using a fresh implementer, risk-based review, fixes, and final verification.
argument-hint: "<plan-file or concrete implementation instruction>"
disable-model-invocation: true
---

# Goal

Implement an explicit actionable plan file or a sufficiently concrete inline implementation instruction requested by the user, with separation between implementation and review. This input authorizes only its described in-scope edits and verification after preflight; it does not authorize handling existing changes, branch changes, destructive actions, or scope expansion.

## Input

$ARGUMENTS

# Role

Act as the orchestrator. Coordinate implementation, independent review, fixes, and final verification. Keep responsibility boundaries clear.

# Rules

* Run immediate preflight before any implementation work.
* Do not stash, discard, commit, or otherwise clean up the user's existing changes on your own.
* Do not create or switch branches without user approval.
* After preflight passes, do not seek per-edit approval for implementation or fixes, but stop at the explicit decision points below. Preflight must pass before any `Edit` or `Write`.
* Do not touch untracked files not created in this session unless the user explicitly asks.
* Use a fresh Implementer subagent for implementation when available.
* Reviewer agents must be read-only and must not fix issues.
* Fixes must be made by an Implementer, not a Reviewer.
* Follow the top-level workflow status and artifact protocol in CLAUDE.md.
* At minimum, report `implementation 開始`, preflight, each implement/review/fix/verification stage, `implementation 保存開始` when a record is written, and exactly one terminal `implementation 完了 summary: ...`, `implementation 中断 summary: ...`, or `implementation 失敗 summary: ...` line.
* For fix/review loops, include the version or round in stage names (for example: `implementation v1 実装開始`, `implementation v1 実装完了 summary: ...`, `implementation v1 review開始`, `implementation v1 review完了 summary: ...`, `implementation v2 修正開始`, `implementation v2 修正完了 summary: ...`).
* After each subagent returns, print the required completion status line first, then (optionally) a short progress note with a few bullets (for an implementer, what it changed; for a reviewer, its verdict and top findings; for the finding-verifier, confirmed/adjusted/rejected counts). Keep it to a handful of lines; do not dump the subagent's full output.
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
   * If the argument is a file path under `<project>/.claude/workflows/...`, derive `<project>` from that path. The project root is the path segment immediately before `.claude/workflows/`.
   * For any other plan file path, use the current Git repository root as `<project>`; if the current directory is not inside a Git repository, use the current working directory and follow the non-Git confirmation rule below.
   * Before interacting about repository state or making edits, verify the current working directory is inside the target project root. If it is not, stop and ask the user to switch to the correct project directory.
   * After project resolution and containment verification, minimally read and validate the supplied plan or inline instruction enough to confirm it exists and is actionable. If it is missing, unreadable, or too vague, stop and ask the user to run `/plan` or provide a clearer instruction. Do not perform deep implementation analysis yet.

   **Tracked-path check (first)**

   * Inspect Git state from the target project root in a way that separates tracked modifications, additions, deletions, renames, and staging from non-ignored untracked paths (for example, `git -C <project> status --short`).
   * Ignored paths—including `.claude/workflows/` when excluded via `.gitignore`, `.git/info/exclude`, or global exclude—remain omitted and do not enter either preflight decision.

   If any tracked path is dirty:

   * stop immediately and briefly identify the tracked paths
   * do not inspect untracked-path relevance, ask either preflight question, launch subagents, or mutate files
   * do not stash, discard, commit, delete, move, or otherwise clean anything on your own
   * end the current run with `implementation 中断 summary: ...`

   **Untracked-path check (second)**

   Run this stage only after the tracked-path check passes. Consider all remaining non-ignored untracked paths, while preserving the rule that files predating this session must not be touched without explicit user instruction.

   First, classify untracked paths under `<project>/.claude/` for a narrow preflight exemption:

   * Never apply the exemption when the target project is `~/.claude` or its resolved path is `/home/kawagutt/.claude`.
   * For any other project, use only clear read-only evidence—such as tracked repository structure, documentation, and instructions—to determine whether the repository develops `.claude` configuration, skills, agents, or related tooling.
   * Exempt these paths only when the evidence clearly shows an ordinary project whose purpose is not `.claude` development. If purpose is ambiguous, do not exempt them.
   * The exemption only prevents those paths from blocking preflight or appearing in the question. It never authorizes reading beyond the evidence needed for classification or modifying the paths.
   * If exempt `.claude/` paths coexist with other untracked paths, continue this stage with only the ordinary paths.

   If no ordinary untracked paths remain, continue to the branch decision. Otherwise, inspect only the read-only evidence needed to assess each path's relevance: location, name or type, relation to plan scope, and project conventions. Present a concise concrete path summary, a recommended action, and why the paths appear unrelated, project-relevant, disposable, or uncertain. For uncertain relevance, explain the uncertainty and conservatively recommend organizing the files.
   Then use the actual `AskUserQuestion` tool with one **single-select** question whose option labels are exactly:

   1. `Continue and leave files untouched`
   2. `Stop so I can commit or organize them`
   3. `Stop so I can delete or move them`
   4. `Cancel implementation`

   Only `Continue and leave files untouched` permits preflight to continue, and it does not authorize touching the listed files. Either stop option or cancellation ends the run with `implementation 中断 summary: ...`. Never clean, delete, move, commit, stash, or otherwise modify the files.

   **Branch decision (after tracked and untracked checks)**

   Run branch analysis only after both Git preflight stages pass. Use the minimal plan validation already performed to classify task size and risk; do not do deep implementation analysis or launch subagents before this decision is resolved.

   * State that preflight passed.
   * Recommend a new branch for non-trivial, multi-file, or behavior-changing work; recommend the current branch for a very small localized fix.
   * Name the recommendation and give the concrete size or risk reason.
   * Use the actual `AskUserQuestion` tool with one **single-select** question whose option labels are exactly:
     1. `Create a new branch`
     2. `Continue on the current branch`
     3. `Cancel`
   * Create or switch branches only after `Create a new branch` is selected. Proceed in place only after `Continue on the current branch` is selected. Do not infer approval from the recommendation, silence, or prior context.
   * If `Cancel` is selected, end the run with `implementation 中断 summary: ...` without branch action.

   If this is not a Git repository, keep the separate confirmation behavior: continue only after the user explicitly confirms how to proceed.

2. Choose checkpoint size.

   * Small task: implement all planned tasks, then review once.
   * Larger task: implement natural checkpoints and review each checkpoint.
   * Do not default to per-task multiple reviewers when that would be unnecessarily heavy.

3. Launch a fresh Implementer subagent.

   Use the `implementer` custom subagent when available. Provide:

   * the explicit actionable plan or inline implementation instruction
   * current repository constraints, including which tracked paths were created by this workflow and remain in scope for a fix round
   * instruction to use `implementation-execution` and `implementation-tdd` practices
   * scope boundaries
   * For a fresh fix Implementer, re-check status first. Stop if unexpected dirty paths appeared; otherwise identify the prior workflow-owned dirty paths that it may continue editing.

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

   * explicit actionable plan or concrete inline instruction
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

   Follow the shared artifact defaults in CLAUDE.md, with these `/implementation`-specific rules:

   * Save the record under the same session directory as the plan. If the plan came from a saved `plan_*.md`, reuse that directory; otherwise derive and create `<project>/.claude/workflows/<yyyymmdd>_<short-slug>/` as in `/plan`.
   * Write the record to the smallest unused `implementation_v<N>.md`; after that succeeds, update `implementation_latest.md` with the same content.
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
