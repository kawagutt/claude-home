---
name: plan
description: Orchestrate creation and independent review of an implementation plan from a shaped instruction or user request.
argument-hint: "<shaped instruction or task>"
disable-model-invocation: true
---

# Goal

Create a practical implementation plan for the user's task, then have it reviewed independently before presenting it to the user.

## Input

$ARGUMENTS

# Role

Act as the orchestrator. Do not implement the task. Coordinate a Planner subagent and a Plan Reviewer subagent, then synthesize the result for the user.

# Rules

* Do not edit repository files.
* Keep all investigation read-only.
* The only file you write is the final plan artifact under `~/.claude/plans/`, outside the target repository, as described in the save step. Do not modify any repository files.
* Prefer the simplest correct plan.
* Preserve fail-fast behavior; do not plan silent fallback behavior or hidden normalization unless requested.
* Do not ask the user questions whose answers can be found by inspecting the repository.
* Ask only for decisions that materially affect the plan.
* The planner and reviewers must be separate contexts.
* The reviewers must not modify the plan or repository.
* Limit planner revision loops to at most two unless the user explicitly asks for more.

# Process

1. Understand the input.

   * If it is too vague to plan, clarify with the user first.
   * If it came from `/shape`, preserve the shaped instruction as the source of truth.

2. Inspect relevant repository context when needed.

   * Look for existing patterns, tests, docs, and constraints.
   * Keep this read-only.

3. Launch a Planner subagent.

   Use the `planner` custom subagent when available. Provide:

   * the original user request or shaped instruction
   * relevant repository context discovered so far
   * constraints from CLAUDE.md and project instructions
   * explicit instruction to use the `plan-planning` skill

4. Launch two independent Plan Reviewer subagents in parallel after the planner returns.

   Run the same review from two model perspectives, each in a fresh separate context:

   * claude-opus-4-8 perspective: spawn the `plan-reviewer` custom subagent with the model set to the `sonnet` alias (remapped to claude-opus-4-8 in this environment via `ANTHROPIC_DEFAULT_SONNET_MODEL`).
   * gpt-5.5 perspective: spawn the `plan-reviewer` custom subagent with the model set to the `opus` alias (which maps to gpt-5.5 in this environment via `ANTHROPIC_DEFAULT_OPUS_MODEL`).

   Provide each reviewer:

   * the original request or shaped instruction
   * the generated plan
   * relevant repository context
   * explicit instruction to use the `plan-review` skill

   Keep the two reviewers independent; do not let one see the other's findings.

5. If either reviewer reports blocking issues:

   * return the combined findings to the Planner subagent for revision
   * note where the two models agree or disagree, and weigh disagreements on their merits rather than by majority
   * repeat both reviews after revision
   * stop after two revision rounds and present remaining risks clearly

6. Synthesize the final plan.

   * Do not dump raw subagent transcripts.
   * Keep the plan actionable and concise.
   * Include assumptions and unresolved decisions.
   * Identify verification steps.

7. Save the plan artifact.

   * Target directory: `~/.claude/plans/<project>/<yyyymmdd>_<slug>/`, where `<project>` is the basename of the git toplevel (or the current directory when not a git repo), `<yyyymmdd>` is `date +%Y%m%d`, and `<slug>` is a short kebab-case label (2-4 words) derived from the task. Create it with `mkdir -p`.
   * If this plan came from a `/shape` result already saved under `~/.claude/plans/...`, reuse that same directory instead of creating a new one.
   * This location is outside the target repository, so the artifact is never committed and never dirties the project.
   * Choose the next version: the smallest `N` starting at 0 for which `plan_v<N>.md` does not exist. Never overwrite an existing `plan_v<N>.md`.
   * Write the synthesized plan to `plan_v<N>.md`, then write the same content to `plan_latest.md`, overwriting it.
   * This is the only file write in this workflow; the planner and reviewer subagents remain read-only.

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

Briefly state whether each independent plan review passed, required revisions, or still has concerns. Report the claude-opus-4-8 and gpt-5.5 perspectives separately, and call out any material disagreement between them.

## Saved artifact

State the absolute path of the `plan_v<N>.md` you wrote, and note that `plan_latest.md` mirrors it.

## Next step

Tell the user they can run `/implementation` with this plan, or point it at the saved `plan_latest.md`, when ready.
