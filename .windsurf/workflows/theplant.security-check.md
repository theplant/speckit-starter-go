---
description: Run security checks on code following security rules from .specify/memory/security-rules/
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Perform security analysis on code by loading relevant security rules on-demand from `.specify/memory/security-rules/`. This workflow validates code against security best practices derived from penetration testing findings and OWASP guidelines.

## Execution Steps

First, create a task plan using `update_plan`:

```
Call update_plan with:
- explanation: "Security check for <target description>"
- plan: [
    {"step": "Identify check type from user input", "status": "pending"},
    {"step": "Load relevant security rules", "status": "pending"},
    {"step": "Identify target code files", "status": "pending"},
    {"step": "Analyze code against security rules", "status": "pending"},
    {"step": "Generate findings report", "status": "pending"},
    {"step": "Provide fix recommendations", "status": "pending"}
  ]
```

Then execute each step, updating status to `in_progress` before starting and `completed` after finishing.

### Step 1: Identify Check Type

Parse `$ARGUMENTS` to determine:
1. **Check type**: One of `csrf`, `xss`, `upload`, `headers`, `input`, `error`, or `all`
2. **Target**: File path, package name, or endpoint to check

If check type is not specified, infer from target:
- Login/auth handlers → `csrf`
- Template rendering → `xss`
- Upload handlers → `upload`
- Middleware → `headers`
- API handlers → `input`
- Error handling code → `error`

If unclear, default to `all` for comprehensive check.

### Step 2: Load Security Rules

**CRITICAL**: ONLY load the rule file(s) matching the check type. Do NOT load all files for specific checks to save context tokens.

Based on check type from Step 1, use `read_file` tool:

| Check Type | Action |
|------------|--------|
| `csrf` | `read_file(".specify/memory/security-rules/csrf-session-rule.md")` |
| `xss` | `read_file(".specify/memory/security-rules/xss-injection-rule.md")` |
| `upload` | `read_file(".specify/memory/security-rules/file-upload-rule.md")` |
| `headers` | `read_file(".specify/memory/security-rules/http-headers-rule.md")` |
| `input` | `read_file(".specify/memory/security-rules/input-validation-rule.md")` |
| `error` | `read_file(".specify/memory/security-rules/error-handling-rule.md")` |
| `all` | Load ALL rule files above |

After loading rules, extract:
1. Relevant `Principle [NAME]` definitions
2. Code examples (correct vs incorrect)
3. AI Agent Requirements checklist
4. Security checklist items

### Step 3: Identify Target Code

Locate the code to analyze:

1. **If specific file/directory provided**, read that file or scan that directory
2. **If no target specified**, use `grep_search` and `find_by_name` to locate security-sensitive code:

**Search Strategy for Any Codebase:**

```
# Find HTTP handlers (any directory structure)
grep_search("http.HandlerFunc\|ServeHTTP\|func.*http.ResponseWriter")

# Find authentication/login code
grep_search("login\|Login\|auth\|Auth\|password\|Password")

# Find cookie operations
grep_search("SetCookie\|http.Cookie\|cookie")

# Find file upload handlers
grep_search("multipart\|FormFile\|Upload\|upload")

# Find template rendering
grep_search("template.Execute\|template.HTML\|html/template")

# Find SQL operations
grep_search("db.Query\|db.Exec\|gorm\|sql.Open")

# Find middleware
grep_search("func.*http.Handler.*http.Handler\|middleware\|Middleware")

# Find all Go files
find_by_name(Pattern="*.go", SearchDirectory=".")
```

**For Legacy Code**: Do NOT assume standard directory names. Use grep to find patterns regardless of location.

### Step 4: Analyze Code Against Rules

For each target file, check against loaded security rules.

**CSRF & Session Checks** (Principle CSRF_PROTECTION, COOKIE_SECURITY, SESSION_MANAGEMENT):
- [ ] State-changing endpoints have CSRF token validation
- [ ] Login endpoint has CSRF protection
- [ ] Cookies use `SameSite=Strict`, `Secure`, `HttpOnly`
- [ ] Session regeneration after authentication

**XSS & Injection Checks** (Principle OUTPUT_ENCODING, HTML_SANITIZATION, SQL_INJECTION):
- [ ] HTML output uses `html/template` (not `text/template`)
- [ ] User content sanitized with bluemonday or equivalent
- [ ] No `template.HTML()` with unsanitized input
- [ ] SQL uses parameterized queries

**File Upload Checks** (Principle FILE_TYPE_VALIDATION, FILE_SIZE_LIMIT, FILE_STORAGE):
- [ ] File type validated (extension + magic bytes)
- [ ] File size limited server-side
- [ ] Files stored outside web root
- [ ] Random filenames used

**HTTP Headers Checks** (Principle SECURITY_HEADERS, CSP_POLICY, CORS_POLICY):
- [ ] Security headers middleware exists
- [ ] CSP configured
- [ ] CORS not using wildcard with credentials

**Input Validation Checks** (Principle INPUT_VALIDATION, LENGTH_LIMITS, RATE_LIMITING):
- [ ] All inputs validated server-side
- [ ] String length limits enforced
- [ ] Rate limiting on auth endpoints
- [ ] URLs validated before use

**Error Handling Checks** (Principle ERROR_MESSAGES, THIRD_PARTY_ERRORS, PANIC_RECOVERY):
- [ ] Errors don't expose internal details
- [ ] Third-party errors wrapped
- [ ] Panic recovery middleware exists

### Step 5: Generate Findings Report

Output findings in this format:

```markdown
## Security Check Report

**Target**: [file/package/endpoint]
**Check Type**: [csrf/xss/upload/headers/input/error/all]
**Date**: [current date]

### Summary

| Severity | Count |
|----------|-------|
| 🔴 CRITICAL | X |
| 🟠 HIGH | X |
| 🟡 MEDIUM | X |
| 🟢 LOW | X |

### Findings

#### [SEVERITY] Finding Title

**Location**: `file:line`
**Rule**: Principle [PRINCIPLE_NAME]
**Description**: What was found
**Impact**: Potential security impact
**Recommendation**: How to fix
```

### Step 6: Provide Fix Recommendations

For each finding:

1. **Explain the vulnerability** in context
2. **Show the current vulnerable code**
3. **Provide corrected code example**
4. **Reference the specific security principle**

If user requests, implement the fixes directly.

## Severity Guidelines

| Severity | Criteria |
|----------|----------|
| 🔴 CRITICAL | Direct exploitation possible, data breach risk, RCE |
| 🟠 HIGH | Significant security impact, requires user interaction |
| 🟡 MEDIUM | Limited impact, defense-in-depth issues |
| 🟢 LOW | Best practice violations, minimal direct risk |

## AI Agent Requirements

- MUST load security rules from `.specify/memory/security-rules/` before analysis
- MUST NOT skip rule loading even for "obvious" checks
- MUST reference specific Principle names (e.g., `Principle CSRF_PROTECTION`)
- MUST provide severity for each finding (CRITICAL/HIGH/MEDIUM/LOW)
- MUST include code examples in recommendations
- MUST generate checklist status for all applicable rules
- SHOULD offer to implement fixes after report
- MUST NOT modify code without explicit user approval

## See Also

- `/theplant.integration-test` - Write integration tests for security fixes
- `/theplant.bugfix` - Bug fix workflow with reproduction-first debugging
- `/theplant.root-cause-tracing` - Trace security issues to root cause
