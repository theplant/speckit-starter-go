# Security Rules Index

This directory contains security rules based on OWASP categories and findings from penetration testing.

## Available Security Rule Documents

| Document | OWASP Category | Use When |
|----------|----------------|----------|
| `csrf-session-rule.md` | A2:2017 Broken Authentication, A5:2017 Broken Access Control | 检查 CSRF 保护、会话管理、Cookie 安全 |
| `xss-injection-rule.md` | A7:2017 XSS, A1:2017 Injection | 检查 XSS 防护、模板注入、输出编码 |
| `file-upload-rule.md` | A6:2017 Security Misconfiguration | 检查文件上传安全、类型验证、大小限制 |
| `http-headers-rule.md` | A6:2017 Security Misconfiguration | 检查 HTTP 安全头配置（CSP, HSTS, XFO等） |
| `input-validation-rule.md` | A1:2017 Injection, A7:2017 XSS | 检查输入验证、参数校验、边界检查 |
| `error-handling-rule.md` | A3:2017 Sensitive Data Exposure | 检查错误处理、敏感信息泄露 |

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

## Related Findings

These rules are derived from:
- Qor Marketing System Penetration Test Report (2025/12/17)
- OWASP ASVS 4.0 Requirements
- OWASP Top 10 2017/2021
