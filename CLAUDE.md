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

## Top-level workflow protocol

The following protocol applies only when the main context runs `/shape`, `/plan`, or `/implementation`. Child agents follow their role-specific instructions; this protocol does not make them prompt users, save workflow artifacts, or modify Git exclude files.

- Emit major-stage status lines as `<stage> 開始` and `<stage> 完了 summary: <short summary>`. End each run with exactly one terminal protocol line: `<workflow> 完了 summary: ...` for success, `<workflow> 中断 summary: ...` when a user choice stops or cancels the run or a required user decision remains unresolved, or `<workflow> 失敗 summary: ...` for an operational error. Emit the terminal protocol line before the structured final response.
- Store workflow artifacts by default under `<project>/.claude/workflows/<yyyymmdd>_<short-slug>/`, where `<project>` is the current Git repository root or, outside Git, the current working directory. Treat generated artifacts as local-only unless the user explicitly asks to commit them.
- In a Git project, before the first artifact write, check whether `.claude/workflows/` is ignored. If it is not, recommend adding `.claude/workflows/` to `.git/info/exclude` and ask for consent. Never modify `.gitignore` for this setup unless the user explicitly asks. Add the exclude rule only after approval and only when no equivalent rule exists. If the user declines, ask whether to save anyway as untracked or stop; never silently write an unignored artifact. Outside Git, save without Git ignore setup.
- Allocate the smallest unused nonnegative version number and never overwrite a versioned artifact. Update the corresponding `*_latest.md` only after the versioned write succeeds.
- Workflow-local rules override these defaults for artifact schema, mandatory versus optional saving, reuse of an upstream workflow directory, and save-failure behavior.
