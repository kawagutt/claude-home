---
name: plan
description: Orchestrate creation and independent review of an implementation plan from a shaped instruction or user request.
argument-hint: "<shape-file or task>"
disable-model-invocation: true
---

# Goal

Create a practical implementation plan for the user's task, then have it reviewed independently before presenting it to the user.

## Input

$ARGUMENTS

# Role

Act as the orchestrator. Do not implement the task. Coordinate a Planner subagent and a Plan Reviewer subagent, then synthesize the result for the user.

Keep orchestration light: confirm input, launch subagents, integrate results, and save the artifact. Do not perform deep repository investigation yourself — delegate that to the Planner.

# Rules

* Do not edit tracked repository files. Aside from the optional `.git/info/exclude` update described in the save step, the only project-local writes are the plan artifact under `.claude/workflows/`.
* Prefer the simplest correct plan.
* Preserve fail-fast behavior; do not plan silent fallback behavior or hidden normalization unless requested.
* Do not ask the user questions whose answers can be found by inspecting the repository.
* Ask only for decisions that materially affect the plan.
* The planner and reviewers must be separate contexts.
* The reviewers must not modify the plan or repository.
* After each subagent returns, print a short progress note to the user before continuing: a first line naming the subagent that finished, then a few bullet points summarizing its result (for the planner, the plan's shape and scope; for each reviewer, its verdict and top findings). Keep it to a handful of lines; do not dump the subagent's full output.
* Limit planner revision loops to at most two unless the user explicitly asks for more.
* Use one Plan Reviewer by default. Add a second independent reviewer only for high-risk plans.

# Process

1. Understand the input.

   * If the argument is a file path, read that shaped instruction and treat it as the source of truth.
   * If the argument is a file path under `<project>/.claude/workflows/...`, derive `<project>` from that path and verify the current working directory is inside that project before investigating or saving. If it is not, stop and ask the user to switch to the correct project directory.
   * If the argument is inline text, use it directly. Use the current git repository root as `<project>`. If the current directory is not inside a git repository, use the current working directory.
   * Do not auto-select the latest saved shape. `/plan` requires an explicit shape file path or inline instruction.
   * If it is too vague to plan, clarify with the user first.

2. Do only minimal repository context gathering.

   * Skim only what is needed to frame the task for the Planner.
   * Keep this read-only.
   * Leave deep investigation to the Planner subagent.

3. Launch a Planner subagent.

   Use the `planner` custom subagent when available. Provide:

   * the shaped instruction or user request
   * any minimal context already discovered
   * constraints from CLAUDE.md and project instructions
   * explicit instruction to use the `plan-planning` skill

4. After the Planner returns, classify the plan risk.

   High-risk indicators include:

   * multi-file behavior changes
   * public API, schema, or contract changes
   * security-sensitive work
   * data migration or compatibility risk
   * architectural restructuring

   * For normal plans, launch one Plan Reviewer subagent.
   * For high-risk plans, launch two Plan Reviewer subagents in parallel in fresh independent contexts, preferably on different configured models. Do not show either reviewer the other reviewer's findings.

   Use the `plan-reviewer` custom subagent. Provide each reviewer:

   * the original request or shaped instruction
   * the generated plan
   * important repository context and constraints discovered by the Planner
   * relevant files, modules, or tests identified by the Planner
   * explicit instruction to use the `plan-review` skill

5. If any reviewer reports blocking issues:

   * return the combined findings to the Planner subagent for revision
   * if two reviewers ran, note where they agree or disagree and weigh disagreements on their merits rather than by majority
   * repeat review after revision
   * stop after two revision rounds and present remaining risks clearly

6. Synthesize the final plan.

   * Do not dump raw subagent transcripts.
   * Keep the plan actionable and concise.
   * Include assumptions and unresolved decisions.
   * Identify verification steps.

7. Save the plan artifact.

   Store workflow artifacts inside the current project, not under the global `~/.claude` directory.

   Default target directory:

   ```text
   <project>/.claude/workflows/<yyyymmdd>_<short-slug>/
   ```

   Where `<project>` is the git repository root (or the current working directory when not in a git repo), `<yyyymmdd>` is today's date (`date +%Y%m%d`), and `<short-slug>` is a short kebab-case label (2–4 words) derived from the task. Create it with `mkdir -p`.

   If this plan came from a saved shape file under `<project>/.claude/workflows/...`, reuse that same directory instead of creating a new one.

   Generated workflow artifacts are local working artifacts by default and should not be committed unless the user explicitly asks.

   Before writing artifacts, check whether `.claude/workflows/` is ignored by git (from the project root, e.g. `git check-ignore -q .claude/workflows/ .claude/workflows/__check__`).

   If the project is a git repository and `.claude/workflows/` is not ignored, tell the user once:

   ```text
   `.claude/workflows/` is not ignored yet.
   I recommend adding it to `.git/info/exclude` so workflow artifacts stay project-local but do not dirty the repo. Should I add it?
   ```

   Do not modify the repository `.gitignore` unless the user explicitly asks.

   If the user approves, add `.claude/workflows/` to `.git/info/exclude` only if an equivalent ignore rule is not already present. Do not duplicate the entry.

   If the user declines adding `.claude/workflows/` to `.git/info/exclude`, ask whether to:

   1. save anyway and allow `.claude/workflows/` to appear as untracked, or
   2. stop without saving.

   Do not silently save an unignored workflow artifact.

   If the project is not a git repository, write artifacts under the project-local workflow directory without git ignore setup.

   * Choose the next version: the smallest `N` starting at 0 for which `plan_v<N>.md` does not exist. Never overwrite an existing `plan_v<N>.md`.
   * Write the synthesized plan to `plan_v<N>.md`, then write the same content to `plan_latest.md`, overwriting it.
   * Aside from the optional `.git/info/exclude` update described above, this is the only file write in this workflow; the planner and reviewer subagents remain read-only.

# Final output

Return:

## Implementation plan

A numbered or task-based plan that another implementer can follow without needing the planning discussion.

Include:

* goal and scope
* files or areas likely to change
* ordered tasks
* tests and verification
* risks or assumptions

## Review summary

Briefly state whether the plan review passed, required revisions, or still has concerns. If a second high-risk reviewer ran, report both perspectives separately and call out any material disagreement.

## Saved artifact

State the absolute path of the `plan_v<N>.md` you wrote, and note that `plan_latest.md` mirrors it.

## Next step

Tell the user they can run `/implementation <plan-file>` with the saved plan path when ready.
