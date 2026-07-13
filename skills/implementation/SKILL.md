---
name: implementation
description: Orchestrate implementation from an explicit actionable plan or concrete inline instruction using a fresh implementer, risk-based review, fixes, and final verification.
argument-hint: "<plan-file or concrete implementation instruction>"
disable-model-invocation: true
model: opus
---

# Goal

Implement an explicit actionable plan file or a sufficiently concrete inline implementation instruction requested by the user, with separation between implementation and review. This input authorizes only its described in-scope edits and verification after preflight; it does not authorize handling existing changes, branch changes, destructive actions, or scope expansion.

## Input

$ARGUMENTS

# Role

Act as the orchestrator. Coordinate implementation, independent review, fixes, and final verification. Keep responsibility boundaries clear.

# Rules

* Run immediate preflight before any implementation work.
* Do not stage, stash, discard, commit, or otherwise clean up the user's existing changes on your own. Staging is prohibited until implementation, review, fixes, and final verification finish; requested staging is separate post-workflow work.
* Do not create or switch branches without user approval.
* After preflight passes, do not seek per-edit approval for implementation or fixes, but stop at the explicit decision points below. Preflight must pass before any `Edit` or `Write`.
* Do not touch untracked files not created in this session unless the user explicitly asks.
* Use a fresh Implementer subagent for implementation when available.
* Reviewer agents must be read-only and must not fix issues.
* Fixes must be made by an Implementer, not a Reviewer.
* Follow the top-level workflow status and artifact protocol in CLAUDE.md.
* Report workflow-specific preflight, implementation, review, fix, verification, and save stages with versions or rounds; rely on the shared protocol for status syntax and the exactly-one-terminal-line requirement.
* After each subagent returns, print the required completion status line first, then (optionally) a short progress note with a few bullets (for an implementer, what it changed; for a reviewer, its verdict and top findings; for the finding-verifier, confirmed/adjusted/rejected counts). Keep it to a handful of lines; do not dump the subagent's full output.
* Prefer simple, scoped changes.
* Preserve fail-fast behavior.
* Do not add fallback behavior, hidden normalization, or silent auto-correction unless requested.
* Run meaningful verification before declaring completion.
* Save a record of the run under `<project>/.claude/workflows/...` in the same session directory as the plan. Record only what carries signal; skip meaningless or empty records.
* Choose the lightest review profile that fits the change. Do not run every reviewer on every task.
* Keep requested alias, configured resolved model, and observed effective model distinct. Telemetry audits but does not gate configured-diverse routing; never infer observed routing. A concrete override that collapses routes prohibits second-model wording.
* The `model: opus` frontmatter applies only to this invocation turn. Later ordinary turns use the session/default route; do not claim the prior skill route remains active. Skill reinvocation is a restart/reload, not normal resume.

# Process

1. **Immediate preflight.** Resolve and minimally validate the target and actionable input before subagents, deep analysis, status interaction, edits, or writes.

   **Resolve target project and input**

   * If the argument is a file path under `<project>/.claude/workflows/...`, derive `<project>` from the path segment immediately before `.claude/workflows/` and use that file as the plan.
   * For any other plan file path, use the current Git repository root as `<project>`.
   * For inline text, use the current Git repository root as `<project>`.
   * If the resolved target is not inside a Git repository, stop and explain: `This workflow requires a Git repository for baseline capture, change-set reconciliation, and review packaging. Please initialize a Git repository, then run /implementation again.` Do not initialize Git or implement through this workflow.
   * Before repository-state interaction, edits, writes, or subagents, verify that the current working directory is inside the resolved target project root. If it is not, stop and ask the user to switch to the correct project directory.
   * Minimally read and validate the supplied plan or inline instruction enough to confirm that it exists, is readable, and contains actionable implementation work. If it is missing, unreadable, or too vague, stop and ask the user to run `/plan` or provide a clearer instruction. Do not perform deep implementation analysis yet.

   **Tracked-path check (first)**

   In Git, capture exactly `git -C <project> status --porcelain=v1 --untracked-files=all`, then first evaluate every tracked modification, addition, deletion, rename, conflict, and staged entry. Ignored paths remain outside dirty-state checks.

   If any tracked path is dirty:

   * stop immediately and briefly identify the tracked paths
   * do not inspect untracked-path relevance, ask either preflight question, launch subagents, or mutate files
   * do not stage, stash, discard, commit, delete, move, or otherwise clean anything
   * end the run with `implementation 中断 summary: ...`

   **Untracked-path check (second)**

   Run this stage only after the tracked-path check passes. Record all pre-existing non-ignored untracked paths. There is no `.claude/` or home-directory exemption. Fingerprint protected untracked paths without following symlinks: for a regular file, record `lstat` metadata, byte size, and a streaming content hash; for a symlink, record `lstat` metadata and hash the link-target text itself without reading the referenced target; for a directory, fingerprint the individual untracked entries reported by `--untracked-files=all` rather than its mtime. For a FIFO, socket, device, or other special type, stop and ask rather than opening or hashing it.

   * If no non-ignored untracked paths remain, continue to the branch decision.
   * Otherwise inspect only the read-only evidence needed to assess each path's scope relevance: location, name or type, relationship to the actionable input, and project conventions.
   * If every path is clearly unrelated, briefly report them and continue automatically without touching or adding them to the session change set.
   * If any path has uncertain relevance, explain the uncertainty and use the actual `AskUserQuestion` tool with one **single-select** question whose option labels are exactly:
     1. `Continue and leave files untouched`
     2. `Stop so I can commit or organize them`
     3. `Stop so I can delete or move them`
     4. `Cancel implementation`
   * If a path clearly overlaps required implementation scope, stop before implementation, explain the collision, and ask the user to add, commit, move, remove, or otherwise organize it. Require a fresh `/implementation` run afterward.

   Record the clean tracked baseline, all pre-existing non-ignored untracked paths, and each protected untracked path's type, mode, byte size, and content hash, plus approved semantic edit scope and approved output paths. `Continue and leave files untouched` never adds a pre-existing untracked path to the session change set. Never modify a pre-existing untracked path during this workflow.

   **Branch decision (after both preflight checks)**

   Use the minimal input validation already performed to classify task size and risk; do not do deep implementation analysis or launch subagents before this decision is resolved.

   * State that both preflight checks passed.
   * Recommend a new branch for non-trivial, multi-file, behavior-changing, or higher-risk work; recommend the current branch for a very small localized fix.
   * State the recommendation and the concrete size or risk reason.
   * Use the actual `AskUserQuestion` tool with one **single-select** question whose option labels are exactly:
     1. `Create a new branch`
     2. `Continue on the current branch`
     3. `Cancel`
   * Create or switch branches only after `Create a new branch` is selected. Proceed in place only after `Continue on the current branch` is selected. Do not infer approval from the recommendation, silence, or prior context.
   * If `Cancel` is selected, end the run with `implementation 中断 summary: ...` without branch action.

   For Git targets, run the tracked and untracked checks and then the branch decision. Do not require an isolated worktree.

2. **Implement in checkpoints.** Give a fresh Implementer the plan, scope, expected outputs, protected existing untracked paths, and TDD/execution instructions. Require no staging and request touched/created-path reporting only as a supplementary cross-check.

   Immediately before each implementation or fix-round Implementer launch, recapture porcelain status and require an exact match with the accepted snapshot: porcelain status, session-change-set manifest, per-changed-path representation/content hashes, and protected-untracked fingerprints. A matching porcelain status alone is insufficient. Stop before launch if any mismatch appears.

   After the Implementer returns, do not require equality with the prior accepted snapshot. Capture the post-round state, compute and validate the expected round delta against assigned scope, and stop on unexpected, out-of-scope, staged, protected, or uncertain changes. If status or any protected untracked type, mode, byte size, or content hash changes unexpectedly, stop and report the affected paths; do not restore, attribute, or reconcile them automatically. After successful reconciliation, establish the post-round state as the new accepted snapshot and track paths first appearing during the session.

   Build the conservative **session change set** from baseline, protected pre-existing untracked set, post-round status delta, approved semantic scope/output paths, and workflow-start-new paths. Evidence may support inclusion but Git does not prove workflow causality. Do not stop solely because a clearly in-scope delta path was omitted from the Implementer report. Stop on scope escape, protected-path collision, staged state, clear concurrent-change evidence, material review-set uncertainty, or material unresolved report/status contradiction. If staged state appears, stop; never unstage files automatically.

3. **Build one canonical base package per unchanged review round.** Include the request and approved plan, session-change-set manifest and status delta, representation of every changed path, relevant constraints, a concise verification summary, snapshot identifier, status snapshot, and per-represented-file content hashes. Do not hash verification output or invent repository-specific package limits or review caps that are not documented.

   Use a conservative default package budget that stays substantially below the model context limit. The budget is a ceiling, not a target: do not pad the package or include unchanged context merely because capacity remains. Reserve enough capacity to represent every changed text path, then prioritize changed hunks and requested-behavior paths before larger excerpts. Reviewers may inspect further context with `Read`, `Grep`, or `Glob`. If the complete review representation cannot fit within the package budget, do not silently omit changed paths or required hunks and do not claim complete review coverage; ask the user to split the implementation into smaller checkpoints or stop review as incomplete.

   Use `git -C <project> diff --full-index --find-renames` for tracked text; plain `git diff` misses untracked files. For small new text use `git diff --no-index -- /dev/null <new-path>` (differences-found is normal success) or complete labelled content. Ordinary text gets a unified diff. Oversized text gets bounded diff/excerpts plus path, type, byte size, content hash, and truncation notice. Binary gets path, change type, mode, byte size, content hash, and relevant metadata/validation only—never raw binary or binary patches. Never silently omit a manifest path.

   Validate complete coverage, representation types, no protected/unrelated paths, and that the current porcelain status, session-change-set manifest, per-changed-path representation/content hashes, and protected-untracked fingerprints match the accepted snapshot; a matching status alone is insufficient. Stop if any protected untracked fingerprint changes. Every reviewer in an unchanged round receives the same base package; rebuild it after every intentional fix.

4. **Select and run reviewers.** Select the lightest applicable reviewer set using the matrix below. Specialized reviewers use requested alias `opus`. Launch every selected reviewer in a fresh independent context, in parallel when possible. Give every reviewer in the same unchanged round the exact same canonical base package and role-specific dynamic invocation metadata. Collect each reviewer verdict and findings before verification or triage.

   | Change type | Reviewers | Conditional addition |
   | --- | --- | --- |
   | Small localized behavior | spec, test | — |
   | Medium behavior | spec, test | refactor only for new abstraction, duplicated logic, or meaningful naming churn |
   | Structural | spec, architecture, test | refactor only when complexity changed |
   | Configuration/dependency/runtime/permission | environment | combine with applicable set |
   | High risk | holistic requested `sonnet` | when configured routes differ from primary `opus` routes |

   Deduplicate categories. Thus configuration-only is `{spec,test,environment}` and structural configuration/runtime with changed complexity is `{spec,architecture,test,environment,refactor}`. Configured diversity requires different alias resolutions, runtime selection support (including `inherit`), and no concrete collapsing override. Missing telemetry yields `configured second-model review; runtime route unconfirmed`; collapsed routes get no second-model label and holistic is added only as a materially different role.

   If a reviewer returns `Needs info`, inspect its exact request. If it needs only additional read-only repository context and the base package snapshot is unchanged, provide a role-specific supplement and rerun only that reviewer. If repository content, status, diff, verification output, or protected-untracked fingerprint changes, invalidate the affected review round, rebuild the canonical base package, and rerun affected reviews only when the change resulted from an intentional fix. Do not treat `Needs info` as Pass or a normal non-blocking concern. If the requested information is unavailable, report that category as incomplete and do not claim it passed.

   After the reviewer batch returns, revalidate the base package snapshot against repository state, including status, changed-path hashes, and protected-untracked fingerprints, before using findings. If it changed unexpectedly while reviewers were running, discard stale findings, report the changed paths, and stop for user direction. Do not automatically attribute, reconcile, rebuild, or rerun after unexplained concurrent changes.

5. **Verify eligible findings.** Verify only blocking findings, materially disputed findings, and significant high-risk holistic concerns. Before each verifier batch, verify the canonical base package snapshot still matches the repository, including status, changed-path hashes, and protected-untracked fingerprints; after it returns, revalidate again. If it changed unexpectedly while verifiers were running, discard stale verdicts, report the changed paths, and stop for user direction. Do not automatically attribute, reconcile, rebuild, or rerun after unexplained concurrent changes. Pass source role, requested alias, configured resolved model, observed model only if telemetry supplies it, routing request, and the canonical base package through the prompt.

   For a known `opus` source use default `sonnet`; for known `sonnet` explicitly override the verifier to `model: opus`. For collapsed or unknown routes, use default `sonnet` only when safe and call it independent verification. Split mixed-source findings where possible; otherwise use default mixed-source independent wording. Telemetry absence never stops verification; insufficient finding/package context may produce `Needs info`. Preserve false-positive removal, severity adjustment, and sibling-occurrence sweeping.

6. **Triage, fix, and rereview.** Send blocking and agreed meaningful findings to a fresh Implementer; limit loops to two unless the user asks. Narrow rereview is allowed only after a demonstrably local fix. Any broader behavioral, structural, test, environment, complexity, or scope impact restores the full affected-category set and requires a rebuilt package.

7. **Final verification.** Run relevant tests, checks, and practical end-to-end validation. Recapture status, ensure no staging, and report exact failures.

8. **Save the run record.** Reuse the plan workflow directory or derive the shared default. Write the smallest unused `implementation_v<N>.md`; never overwrite an existing versioned record. The workflow explicitly authorizes replacing `implementation_latest.md` with identical content only after the versioned write succeeds. Before replacing an existing `implementation_latest.md`, read it to confirm it is the expected workflow artifact; then use `Edit` or `Write` to update it. Do not ask for separate overwrite consent unless its content or location contradicts that expectation. Include only meaningful review, change, finding, verification, and round information; skip empty records. Preserve shared ignored-artifact and save-failure behavior and never alter ignore/exclude setup implicitly.

# Final output

Return exactly:

## Completed

Summarize what changed.

## Reviews

State the review profile and outcomes, including incomplete categories or verified findings when relevant. When model-diverse review or verification ran, describe routing using requested alias, configured resolved model, and observed effective model when telemetry is available. Omit routing details when no model-diverse launch occurred and there is no routing limitation to report.

## Verification

List commands or checks run and their results.

## Notes

Mention remaining risks, assumptions, or follow-up work only if relevant.

## Saved record

State the versioned record path and that `implementation_latest.md` mirrors it, or say no meaningful record was warranted.
