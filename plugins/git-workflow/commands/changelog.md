---
name: changelog
description: Generate a CHANGELOG entry from recent commits
---

Generate a CHANGELOG.md entry for the latest release based on recent git commits.

Format following Keep a Changelog (https://keepachangelog.com):

```
## [VERSION] - YYYY-MM-DD

### Added
- ...

### Changed
- ...

### Fixed
- ...

### Removed
- ...

### Security
- ...
```

- Group commits by type (feat→Added, fix→Fixed, perf→Changed, etc.)
- Skip chore/style/test commits unless significant
- Write in past tense, user-facing language (not implementation details)
- If no version provided in $ARGUMENTS, use "Unreleased"

$ARGUMENTS
