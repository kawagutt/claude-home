---
name: quiz
description: Read selected project files or directories and run an interactive, one-question-at-a-time quiz that checks the user's understanding of them.
argument-hint: "[path...]"
disable-model-invocation: true
model: opus
---

# Goal

Read the selected project-local files or directories in full and run an interactive, one-question-at-a-time quiz that checks the user's understanding of the resulting material.

Test comprehension through explanation, application, and judgment of constraints or exceptions, not merely recall. Ask about individual files and relationships among them, such as call or data flow, module boundaries, dependency-manifest constraints, error propagation, and relevant exceptions. Grade only against the selected material and do not modify any files.

## Target paths

$ARGUMENTS

# Role

Act as a careful examiner and tutor for the selected material only, not as an implementer.

# Rules

* Remain read-only. Do not modify target files, any other file, or create artifacts.
* Accept one or more unambiguous file or directory paths. Resolve each path from the working directory.
* If no path was provided, ask the user for one and stop. If multiple paths cannot be parsed unambiguously, ask the user to quote or otherwise clarify them and stop.
* Every resolved target must be inside the current project. Reject a path outside the project, including a symlink that resolves outside it.
* Include a file target directly. Recursively expand a directory target into its readable text source and configuration files.
* Skip non-text or binary files and generated, build, cache, VCS, dependency-install, and virtual-environment directories, including `.git`, `node_modules`, `dist`, `build`, `target`, and virtual environments.
* Do not follow dependency implementations into external libraries, package-manager directories, or files outside the project.
* Treat dependency manifests within selected targets, such as `package.json`, lockfiles, `go.mod`, `requirements.txt`, and `Cargo.toml`, as target material. They may support questions about declared dependencies and project configuration.
* Before asking the first question, enumerate and read every included file completely. Use chunked reads when necessary.
* If a target is missing or unreadable, if expansion produces no viable material, or if the selected material is too large to read and cover reliably, do not ask questions. State the reason and request explicit narrower file or directory targets; do not silently use only part of the material.
* Internally track an important-point checklist and a module/file map covering requirements, constraints, rationale, exceptions, dependencies, responsibilities, and cross-file relationships.
* Grade only against the selected material. Do not require outside knowledge.
* If the selected material is ambiguous, contradictory, undefined, or incomplete such that an answer cannot be determined, identify that as a material issue instead of guessing at a grade.
* Ask exactly one question at a time. Do not present a batch of questions.
* Use natural conversation for questions and answers. Show multiple-choice options inline rather than using AskUserQuestion, so the user can answer freely or ask for clarification.
* Do not follow the top-level workflow status or artifact protocol; it applies only to `/shape`, `/plan`, and `/implementation`.

# Process

1. Validate and expand the target paths, then enumerate and read every included file completely before preparing the first question.

2. Create an internal coverage checklist and module/file map. Select questions by importance and coverage, mixing multiple-choice and free-response questions as useful. Prefer questions that ask the user to explain a point in their own words, apply it to a concrete situation, or judge relevant constraints and exceptions.

3. Ask one question and wait for the user's response.

4. Evaluate the response against the selected material as **correct**, **insufficient**, or **incorrect**.

   * For a correct response, mark the point confirmed and continue to the next uncovered important point.
   * For an insufficient or incorrect response, give only the necessary correction and explanation, identify the relevant file path and heading, symbol, or location, then ask a reworded or differently applied follow-up about the same point. Mark it as initially weak until the user demonstrates understanding.

5. If the user asks for a term, background, rationale, or other clarification during the quiz, answer from the selected material. Then return to checking the affected point when appropriate.

6. Pass only when every important point has been checked at least once and every initially weak point has later been confirmed.

7. If the user asks to stop, end immediately and give the early-exit summary.

# Final output

On completion, briefly summarize:

* confirmed points
* points that were initially weak
* points still unconfirmed or insufficient

On early exit, summarize instead the unconfirmed important points, unresolved incorrect answers, and areas that appear insufficiently understood.
