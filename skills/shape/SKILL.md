---
name: shape
description: Clarify a rough idea through focused discussion and turn it into a concise, unambiguous instruction for a later Claude Code task.
argument-hint: "<rough idea or request>"
disable-model-invocation: true
---

# Goal

Help the user clarify the following rough idea and turn it into a concise, clear instruction that can later be given to Claude Code or another agent.

This is not a specification or implementation-planning skill. The output is a short, handoff-ready instruction.

## Rough input

$ARGUMENTS

# Role

Act as a thoughtful technical discussion partner, not as an implementer or specification writer.

The user's initial idea may be incomplete, based on a misunderstanding, or not yet the best solution. Help improve the underlying idea before polishing the wording.

# Rules

* Do not implement anything.
* Do not write a detailed specification or implementation plan unless the user explicitly asks for one.
* Do not edit tracked repository files. Aside from the optional `.git/info/exclude` update described in the save step, the only project-local writes are the shaped-instruction artifact under `.claude/workflows/`.
* Always attempt to save the final shaped instruction when the discussion is complete. If saving is blocked by the user's decision or a filesystem error, paste the shaped instruction in the chat so it is not lost.
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

8. Save the shaped instruction as described below.

# Saved artifact

Attempt to save every completed `/shape` run so a later `/plan` can reuse it.

## Artifact location

Store workflow artifacts inside the current project, not under the global `~/.claude` directory.

Default target directory:

```text
<project>/.claude/workflows/<yyyymmdd>_<short-slug>/
```

Where `<project>` is the git repository root (or the current working directory when not in a git repo), `<yyyymmdd>` is today's date (`date +%Y%m%d`), and `<short-slug>` is a short kebab-case label from the topic. Create it with `mkdir -p`.

Resolve `<project>` before saving. Use the current git repository root when inside a git repository; otherwise use the current working directory.

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

## Files

* Choose the smallest `N` starting at 0 for which `shape_v<N>.md` does not exist. Never overwrite an existing `shape_v<N>.md`.
* Write the full shaped instruction to `shape_v<N>.md`, then write the same content to `shape_latest.md`, overwriting it.
* Include a short YAML frontmatter block with at least `status: final` and `created: <ISO-8601 timestamp>`.
* If the discussion ends before the instruction is ready, you may save `shape.draft.md` with `status: draft` instead. Do not mark a draft as final.
* `/plan` must receive an explicit shape file path. Do not auto-select `shape_latest.md` or any other saved shape.

# Final output

After saving, return only:

## Summary

A short summary of the shaped instruction in a few sentences.

## Saved artifact

* Absolute path to the `shape_v<N>.md` you wrote, and note that `shape_latest.md` mirrors it
* Reminder to run `/plan <shape-file>` with that explicit path

Do not paste the full shaped instruction into the chat when saving succeeds. The saved file is the handoff artifact.

If saving fails, do not silently drop the shaped instruction. Report the failure and paste the shaped instruction in the chat so the user can copy it.

The saved `shape_v<N>.md` must contain:

## Shaped instruction

A concise, self-contained instruction that the user can give directly to Claude Code.

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

## Notes

Include only unresolved assumptions or a very brief explanation of an important decision. Omit this section when there is nothing useful to add.
