---
name: pr
description: Draft a pull request title and description from branch commits
---

Generate a pull request description based on the commits between this branch and main/master.

Structure the PR description as:

```
## Summary
<2-3 sentence overview of what this PR does and why>

## Changes
- <bullet list of meaningful changes>

## Testing
- <how to test / what was tested>

## Screenshots / Demo
<if UI changes, note where to add screenshots>

## Checklist
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] No breaking changes (or breaking changes documented)
```

Also suggest a concise PR title (under 70 chars) following the branch/commit context.

$ARGUMENTS
