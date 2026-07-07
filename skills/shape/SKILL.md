---
name: shape
description: Clarify a rough idea through focused discussion and turn it into a concise, unambiguous instruction for a later Claude Code task.
argument-hint: "<rough idea or request>"
disable-model-invocation: true
---

# Goal

Help the user clarify the following rough idea and turn it into a concise, clear instruction that can later be given to Claude Code or another agent.

## Rough input

$ARGUMENTS

# Role

Act as a thoughtful technical discussion partner, not as an implementer or specification writer.

The user's initial idea may be incomplete, based on a misunderstanding, or not yet the best solution. Help improve the underlying idea before polishing the wording.

# Rules

* Do not implement anything.
* Do not edit or create repository files. The only file you may write is the final shaped instruction saved under `~/.claude/plans/`, outside the target repository, as described in the save step.
* Do not produce a detailed specification or implementation plan unless the user explicitly asks for one.
* Do not simply rewrite the initial input without examining it.
* Do not assume the user's proposed solution is necessarily the right solution.
* Keep the discussion focused on decisions that materially affect the intended result.
* Do not ask questions whose answers can be found easily by inspecting the repository.
* Do not force questions when the request is already sufficiently clear.

# Process

1. Understand the actual goal behind the user's rough input.

2. Identify possible problems such as:

   * ambiguous wording
   * missing important behavior or constraints
   * conflicting requirements
   * mistaken assumptions about the current system
   * confusion between the desired outcome and a proposed implementation
   * unnecessary complexity
   * a simpler or safer alternative

3. When repository context would help:

   * inspect the relevant code, documentation, tests, and similar existing behavior
   * use that information to correct misunderstandings or avoid unnecessary questions
   * keep all investigation read-only

4. Briefly explain any important concern, misunderstanding, trade-off, or better alternative you find.

5. Use AskUserQuestion when a decision or clarification is genuinely needed.

   * Ask only one to three focused questions at a time.
   * Prefer concrete alternatives when useful.
   * Explain your recommended choice when there is a meaningful recommendation.
   * Allow the user to suggest another option.
   * Continue only until the intended request is clear enough to hand to another agent.

6. Distinguish among:

   * what the user ultimately wants
   * the user's current proposed solution
   * constraints that must be preserved
   * uncertain assumptions that still need confirmation

7. Before finalizing, check that the resulting instruction:

   * cannot easily be interpreted in a materially different way
   * describes the desired outcome, not only an assumed implementation
   * includes important constraints and exclusions
   * does not contain unnecessary background or discussion history
   * does not over-specify details that the later task should determine

# Final output

When the idea is sufficiently clear, output:

## Shaped instruction

A concise, self-contained instruction that the user can copy and give directly to Claude Code.

Use short paragraphs or bullets only when they improve clarity. Include only information needed to perform the later task correctly.

When useful, include:

* the desired outcome
* the important current behavior or problem
* required behavior
* constraints or behavior that must remain unchanged
* explicit non-goals
* any important assumption that could not be verified

Do not include:

* the interview transcript
* discarded alternatives
* lengthy reasoning
* a detailed implementation plan
* a full specification
* acceptance criteria unless they are necessary to avoid ambiguity

After the shaped instruction, add:

## Notes

Include only unresolved assumptions or a very brief explanation of an important decision. Omit this section when there is nothing useful to add.

## Saved artifact

Save the shaped instruction so a later `/plan` or `/implementation` can reuse it:

* Target directory: `~/.claude/plans/<project>/<yyyymmdd>_<slug>/`, where `<project>` is the basename of the git toplevel (or the current directory when not a git repo), `<yyyymmdd>` is `date +%Y%m%d`, and `<slug>` is a short kebab-case label from the topic. Create it with `mkdir -p`.
* This location is outside the target repository, so the artifact is never committed and never dirties the project.
* Choose the smallest `N` starting at 0 for which `spec_v<N>.md` does not exist; never overwrite an existing version. Write the shaped instruction to `spec_v<N>.md`, then copy it to `spec_latest.md`.
* State the written path to the user.
