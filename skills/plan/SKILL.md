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
* Prefer the simplest correct plan.
* Preserve fail-fast behavior; do not plan silent fallback behavior or hidden normalization unless requested.
* Do not ask the user questions whose answers can be found by inspecting the repository.
* Ask only for decisions that materially affect the plan.
* The planner and reviewer must be separate contexts.
* The reviewer must not modify the plan or repository.
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

4. Launch a Plan Reviewer subagent after the planner returns.

   Use the `plan-reviewer` custom subagent when available. Provide:

   * the original request or shaped instruction
   * the generated plan
   * relevant repository context
   * explicit instruction to use the `plan-review` skill

5. If the reviewer reports blocking issues:

   * return the review to the Planner subagent for revision
   * repeat review after revision
   * stop after two revision rounds and present remaining risks clearly

6. Synthesize the final plan.

   * Do not dump raw subagent transcripts.
   * Keep the plan actionable and concise.
   * Include assumptions and unresolved decisions.
   * Identify verification steps.

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

Briefly state whether the independent plan review passed, required revisions, or still has concerns.

## Next step

Tell the user they can run `/implementation` with this plan when ready.
