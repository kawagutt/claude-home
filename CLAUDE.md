# CLAUDE.md

## Working rules

- Do not run history-changing or remote Git operations unless the user explicitly asks.
- Read-only exploration is pre-approved: inspect files under the current project, and run read-only commands such as `git status`, `git diff`, `git log`, `rg`, `ls`, and `cat` without asking first. Prefer scoped read-only commands over dumping large logs or full diffs when a narrower command is sufficient.
- File edits and writes under the current project are pre-approved during normal implementation work after the working tree preflight passes. Do not ask before each `Edit` or `Write` when executing an approved plan or an explicit fix request. This does not override the dirty-worktree rule, the untracked-file rule, or the ban on destructive/history-changing operations.
- Before starting a new non-trivial or multi-file edit, check whether relevant tracked files are already dirty. If they are, stop and ask instead of mixing unrelated edits.
- Do not touch untracked files that were not created in the current session unless the user explicitly asks.
- Preserve fail-fast behavior. Do not add fallback behavior, hidden normalization, or silent auto-correction unless requested.
- Prefer the simplest correct change. Avoid unnecessary abstractions, compatibility shims, and drive-by refactors.
- Comments should be minimal and in English.

