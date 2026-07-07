# CLAUDE.md

## Working rules

- Do not run history-changing or remote Git operations unless the user explicitly asks.
- Before starting a new non-trivial or multi-file edit, check whether relevant tracked files are already dirty. If they are, stop and ask instead of mixing unrelated edits.
- Do not touch untracked files that were not created in the current session unless the user explicitly asks.
- Preserve fail-fast behavior. Do not add fallback behavior, hidden normalization, or silent auto-correction unless requested.
- Prefer the simplest correct change. Avoid unnecessary abstractions, compatibility shims, and drive-by refactors.
- Comments should be minimal and in English.

