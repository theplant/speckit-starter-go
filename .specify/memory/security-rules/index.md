# Security Rules Index

This directory contains security rules based on OWASP ASVS 5.0 and qor marketing penetration testing findings.

## Available Security Rule Documents

| Document | ASVS Chapters | Use When |
|----------|---------------|----------|
| `csrf-session-rule.md` | V3, V6, V7 | CSRF, Cookie, Session, Authentication, Password |
| `xss-injection-rule.md` | V1 | XSS, SQL/LDAP/Command Injection, Deserialization |
| `file-upload-rule.md` | V5 | File Upload, Type Validation, Path Traversal, Archive |
| `http-headers-rule.md` | V3, V4, V8, V15 | Security Headers, CSP, CORS, TLS, JWT, API |
| `input-validation-rule.md` | V2, V9 | Input Validation, Business Logic, Authorization, Access Control |
| `error-handling-rule.md` | V11, V14, V16 | Error Handling, Logging, Data Protection, Cryptography |

## Rule Document Structure

Each rule document follows a consistent structure:

```markdown
# [Rule Category] Security Rules

## Principle [PRINCIPLE_NAME]. [Title]

- Rule 1 with MUST/MUST NOT/SHOULD
- Rule 2 with specific requirements
- ...

**Rationale**: Why this rule exists

**ASVS Requirements**: Related OWASP ASVS requirements

**Examples**: Code examples (correct vs incorrect)

**AI Agent Requirements**: Specific checks AI must perform
```

## How to Load Rules

Rules are loaded on-demand by the `theplant.security-check` workflow based on the check type:

```
/theplant.security-check csrf     → loads csrf-session-rule.md
/theplant.security-check xss      → loads xss-injection-rule.md
/theplant.security-check upload   → loads file-upload-rule.md
/theplant.security-check headers  → loads http-headers-rule.md
/theplant.security-check input    → loads input-validation-rule.md
/theplant.security-check error    → loads error-handling-rule.md
/theplant.security-check all      → loads all rules
```

## Related Standards

These rules are derived from:
- **OWASP ASVS 5.0** (May 2025) - Primary reference
- Qor Marketing System Penetration Test Report (2025/12/17)
- OWASP Top 10 2021

## ASVS 5.0 Chapter Coverage

| ASVS Chapter | Rule File | Coverage |
|--------------|-----------|----------|
| V1 Encoding/Sanitization | `xss-injection-rule.md` | ✅ |
| V2 Validation/Business Logic | `input-validation-rule.md` | ✅ |
| V3 Web Frontend | `http-headers-rule.md` | ✅ |
| V5 File Handling | `file-upload-rule.md` | ✅ |
| V6 Authentication | `csrf-session-rule.md` | ✅ |
| V7 Session Management | `csrf-session-rule.md` | ✅ |
| V8 Self-contained Tokens | `http-headers-rule.md` | ✅ |
| V9 Authorization | `input-validation-rule.md` | ✅ |
| V11 Cryptography | `error-handling-rule.md` | ✅ |
| V14 Data Protection | `error-handling-rule.md` | ✅ |
| V15 Secure Communication | `http-headers-rule.md` | ✅ |
| V16 Logging/Error Handling | `error-handling-rule.md` | ✅ |
