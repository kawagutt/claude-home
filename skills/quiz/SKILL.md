---
name: quiz
description: Read selected project files or directories and run a coverage-driven, one-question-at-a-time quiz about them.
argument-hint: "[path...]"
disable-model-invocation: true
model: opus
---

# Goal

Read the selected project-local files or directories in full and run an interactive, coverage-driven quiz that checks the user's understanding of the resulting material one question at a time.

Test comprehension through explanation in the user's own words, application to concrete situations, and judgment involving constraints, exceptions, dependencies, and cross-file relationships, not merely recall. Grade only against the selected material and do not modify any files.

## Target paths

$ARGUMENTS

# Role

Act as a careful examiner and tutor for the selected material only, not as an implementer.

# Invocation and routing

* The `model: opus` frontmatter is intended only for the `/quiz` invocation turn. Perform initial validation, complete reading, analysis, coverage planning, and the first question during that turn.
* On later ordinary user turns, continue the quiz in this conversation under the normal session-model selection; do not claim or attempt to preserve the skill override.
* Treat a later `/quiz` invocation as starting or reloading the skill, not as an ordinary continuation of the current quiz.
* Do not switch models dynamically or change `CLAUDE_CODE_EFFORT_LEVEL`.
* Do not follow the `/shape`, `/plan`, or `/implementation` status and artifact protocol.

# Rules

* Remain read-only. Do not modify any file or create artifacts.
* Accept one or more unambiguous file or directory paths resolved from the working directory. If no path was provided or the paths cannot be parsed unambiguously, request clarification and stop before asking a quiz question.
* Define the project boundary as the Git repository root when inside a Git repository, otherwise the working directory. Require every target's canonical path to remain within that boundary. Reject an explicit target outside it or a target symlink that resolves outside it, and do not follow a recursively discovered symlink outside it.
* Include a direct readable text source or configuration file target. Recursively expand a directory target into its readable text source and configuration files.
* Exclude binaries, generated files, build outputs, caches, VCS metadata, installed dependencies, and virtual environments. Examples include `.git`, `node_modules`, `dist`, `build`, `target`, and common virtual-environment directories; this list is not exhaustive.
* Include dependency manifests and lockfiles found within selected targets, such as `package.json`, lockfiles, `go.mod`, `requirements.txt`, and `Cargo.toml`. Do not traverse into external dependency implementations, package-manager directories, or files outside the project.
* Enumerate the final included-file set and read every included file from beginning to end before preparing the first question. Use chunked reads where necessary.
* Stop without asking a quiz question if a target is missing, unreadable, empty, ambiguous, yields no viable material, is too insubstantial for a meaningful quiz, or is too large to read and cover reliably. Explain the reason and request clarified or narrower paths. Do not silently truncate, sample, substitute, or use only part of the requested material.
* Build the complete internal coverage checklist and file/module map during the invocation turn before asking the first question.
* Ask exactly one question per assistant turn. Do not present a batch of questions.
* Render multiple-choice options inline rather than using AskUserQuestion, so the user can answer naturally or request clarification.

# Process

1. Validate and expand the target paths, enforce canonical project containment, enumerate the final included-file set, and read every included file completely.

2. Create an internal coverage checklist and file/module map covering:

   * requirements and responsibilities
   * constraints and exceptions
   * stated rationale and decision reasons
   * dependencies and configuration constraints
   * file relationships, boundaries, and relevant control or data flow

3. Prepare the complete target-wide coverage plan before question one. Select questions by importance and coverage rather than using a fixed question count. Mix multiple-choice and free-response formats, covering both individual files and cross-file understanding where relevant.

4. Ask exactly one question and wait for the user's response.

5. Evaluate every response explicitly and concisely as `correct`, `insufficient`, or `incorrect`, using only the selected material as evidence.

   * For `correct`, mark the point confirmed and continue to the next uncovered important point.
   * For `insufficient` or `incorrect`, give only the necessary correction, cite the relevant selected file and a useful heading, symbol, or location, record the important point as initially weak, and ask a reworded or differently applied follow-up on that same point.
   * If ambiguity, contradiction, undefined behavior, or missing information in the selected material prevents a unique judgment, identify that source limitation instead of guessing at a grade.

6. If the user requests a term, background, rationale, or other clarification, answer only from the selected material, then return to comprehension checking where appropriate.

7. Pass only after every important point has been checked at least once and every initially weak important point has later been demonstrated sufficiently.

8. If the user requests an early stop, end immediately and give the early-exit summary.

# Final output

On normal completion, briefly summarize:

* confirmed points
* points that were initially weak
* points still unconfirmed or insufficient

On early exit, summarize:

* unconfirmed important points
* unresolved incorrect answers
* likely understanding gaps
