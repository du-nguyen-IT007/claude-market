---
name: security
description: Run a security-focused audit on the current file
---

Perform a security audit on this code. Check for OWASP Top 10 vulnerabilities and common issues:

- SQL/NoSQL injection
- XSS (Cross-Site Scripting)
- Insecure deserialization
- Hardcoded secrets or credentials
- Insecure direct object references
- Missing authentication/authorization checks
- Sensitive data exposure
- Insecure cryptography usage
- Command injection
- Path traversal

Report each finding with:
- Vulnerability type
- CWE ID (if applicable)
- Affected line(s)
- Risk level: `CRITICAL / HIGH / MEDIUM / LOW`
- Remediation steps

$ARGUMENTS
