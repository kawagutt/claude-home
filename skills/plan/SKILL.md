---
name: plan
description: Orchestrate creation and independent review of an implementation plan from a shaped instruction or user request.
argument-hint: "<shape-file or task>"
disable-model-invocation: true
model: opus
---

# Goal

Create a practical implementation plan for the user's task, then have it reviewed independently before presenting it to the user.

## Input

$ARGUMENTS

# Role

Act as the orchestrator. Do not implement the task or author substantive plan content. Coordinate a Planner subagent and independent Plan Reviewer subagent(s), route findings back to the Planner for revision, select the reviewed Planner result, then report, save, and present it. You may format the plan without materially changing it; return substantive changes to the Planner.

Keep orchestration light: confirm input, launch subagents, integrate results, and save the artifact. Do not perform deep repository investigation yourself — delegate that to the Planner.

# Rules

* Do not edit tracked repository files. Aside from the optional `.git/info/exclude` update described in the save step, the only project-local writes are the plan artifact under `.claude/workflows/`.
* Follow the top-level workflow status and artifact protocol in CLAUDE.md.
* Do not ask the user questions whose answers can be found by inspecting the repository.
* Ask only for decisions that materially affect the plan.
* The planner and reviewers must be separate contexts.
* The reviewers must not modify the plan or repository.
* Report workflow-specific planner, reviewer, revision, and save stages with the plan version or round; rely on the shared protocol for status syntax and the exactly-one-terminal-line requirement.
* The `model: opus` frontmatter applies only to the current invocation turn. On a later ordinary user turn, report the session/default route; do not claim the prior skill route remains active. Treat skill reinvocation as a restart/reload, not a normal resume.
* After each subagent returns, print the required completion status line first, then (optionally) a short progress note to the user with a few bullets (for the planner, the plan's shape and scope; for each reviewer, its verdict and top findings). Keep it to a handful of lines; do not dump the subagent's full output.
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

4. After the Planner returns, classify the plan risk holistically.

   Risk indicators include:

   * multi-file scope
   * behavior changes
   * public API, schema, or contract changes
   * security-sensitive work
   * data migration or compatibility risk
   * architectural restructuring

   Treat these as inputs to an overall judgment, not automatic escalators.

   * For normal plans, launch one fresh independent primary Plan Reviewer using the `plan-reviewer` agent (requested alias `opus`).
   * For high-risk plans, retain that primary reviewer and launch a second fresh independent Plan Reviewer explicitly with `model: sonnet`, in parallel. Give both identical request, plan, and repository context without sharing findings.


   Routing terminology and evidence:

   * Keep the requested alias (`opus` or `sonnet`), configured resolved model from settings, and observed effective model from runtime telemetry distinct. Never infer observed routing from aliases or settings.
   * Configured diversity exists only when the aliases resolve differently, the installed runtime supports the selection (including `inherit` behavior), and no concrete override collapses the routes. Telemetry audits the launch; it is not a gate. Without telemetry, report `configured second-model review; runtime route unconfirmed`.
   * If a concrete override collapses both routes, do not use second-model wording. Add another reviewer only for a materially different role and call the result role-diverse.

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

6. Select and present the final Planner-authored plan.

   * Do not dump raw subagent transcripts.
   * Ensure the selected plan is actionable and concise; return any material content changes to the Planner.
   * Include assumptions and unresolved decisions.
   * Identify verification steps.

7. Save the plan artifact.

   Follow the shared artifact defaults in CLAUDE.md, with these `/plan`-specific rules:

   * Use `<project>/.claude/workflows/<yyyymmdd>_<short-slug>/`, with today's date (`date +%Y%m%d`) and a short kebab-case task label; create it with `mkdir -p`.

   * If this plan came from a saved shape file under `<project>/.claude/workflows/...`, reuse that same directory instead of creating a new one.
   * Write the selected Planner-authored plan to the smallest unused `plan_v<N>.md`; never overwrite an existing versioned plan. The workflow explicitly authorizes replacing `plan_latest.md` with identical content only after the versioned write succeeds. Before replacing an existing `plan_latest.md`, read it to confirm it is the expected workflow artifact; then use `Edit` or `Write` to update it. Do not ask for separate overwrite consent unless its content or location contradicts that expectation.
   * The allowed project-local artifact write set is those two plan files. A separately consented `.git/info/exclude` update is also allowed by the shared protocol; planner and reviewer subagents remain read-only.

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
