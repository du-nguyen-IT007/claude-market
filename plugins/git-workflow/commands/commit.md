---
name: commit
description: Generate a conventional commit message from staged changes
---

Analyze the current staged changes (`git diff --cached`) and generate a commit message following the Conventional Commits specification:

Format: `<type>(<scope>): <short description>`

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`, `ci`, `build`, `revert`

Rules:
- Subject line max 72 chars, imperative mood, no period at end
- Add a blank line then body if changes need explanation
- Reference issue numbers if detectable from branch name or file context
- If multiple logical changes exist, suggest splitting into separate commits

Output the commit message ready to copy, then briefly explain the type/scope choices.

$ARGUMENTS
