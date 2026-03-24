---
name: review
description: Perform a thorough code review of the current file or selection
---

Review the provided code thoroughly. Focus on:

1. **Correctness** — logic errors, edge cases, off-by-one errors
2. **Security** — injection risks, insecure defaults, exposed secrets
3. **Performance** — unnecessary loops, N+1 queries, memory leaks
4. **Readability** — naming, complexity, dead code
5. **Best practices** — language idioms, framework conventions

For each issue found, output:
- Severity: `[CRITICAL | HIGH | MEDIUM | LOW | INFO]`
- Location: file and line number
- Issue description
- Suggested fix with code snippet

End with a summary score out of 10 and top 3 action items.

$ARGUMENTS
