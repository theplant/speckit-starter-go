# Error Handling Security Rules

## Principle ERROR_MESSAGES

**Requirements**:
- Production → generic messages only
- NEVER expose: stack traces, SQL queries, file paths
- Log details server-side
- Include trace_id for support correlation

**Detection Pattern**:
```go
// ❌ http.Error(w, fmt.Sprintf("DB error: %v", err), 500)
// ✅ handleError(err) + return {"code":"ERROR", "message":"An error occurred"}
```

## Principle THIRD_PARTY_ERRORS

**Requirements**:
- NEVER expose raw third-party API errors
- Log original error internally
- Return generic application error

**Detection Pattern**:
```go
// ❌ return fmt.Errorf("LINE error: %v, body: %s", err, resp.Body)
// ✅ handleError(err) + return ErrExternalService
```

## Principle PANIC_RECOVERY

**Requirements**:
- Recovery middleware → catch all panics
- Log panic + stack trace
- Return generic 500 error
- Prevent single request crashing server

**Detection**: Look for `defer func() { recover() }` middleware.

## Principle LOGGING_SECURITY

**NEVER Log**:
- Passwords
- API keys / tokens
- Session IDs
- Credit card numbers

**Detection Pattern**:
```go
// ❌ log.Printf("password: %s", password)
// ✅ Mask: headers["Authorization"] = "[REDACTED]"
```

## Principle SECURITY_LOGGING (ASVS 16.3)

**Requirements**:
- Log ALL authentication attempts (success + failure)
- Log authorization failures
- Log input validation failures
- Include: timestamp, user ID, IP, action, result
- Structured format (JSON) preferred

**Detection Pattern**:
```go
// ❌ No logging/tracing on auth failure
// ✅ log/trace: ("auth_attempt", "user", email, "success", false, "ip", ip)
```

## Principle LOG_PROTECTION (ASVS 16.4)

**Requirements**:
- Logs → encode to prevent injection
- Logs → protected from modification
- Logs → transmitted to separate system
- Retain logs per compliance requirements

## Principle DATA_PROTECTION (ASVS 14.1)

**Requirements**:
- Classify data sensitivity levels
- PII → encrypted at rest and in transit
- Secrets → use vault/secrets manager (NOT env vars in code)
- Memory → clear sensitive data after use

**Detection Pattern**:
```go
// ❌ Hardcoded secrets
apiKey := "sk-1234567890"

// ✅ From secrets manager
apiKey := os.Getenv("API_KEY") // or vault.Get("api-key")
```

## Principle CRYPTOGRAPHY (ASVS 11.2)

**Requirements**:
- Use approved algorithms: AES-GCM, ChaCha20-Poly1305
- Key length → min 128 bits (256 recommended)
- Random → use crypto/rand (NOT math/rand)
- NO: ECB mode, MD5, SHA1 for security, DES

**Detection Pattern**:
```go
// ❌ math/rand for tokens, ECB mode, MD5
// ✅ crypto/rand.Read(), AES-GCM, SHA-256+
```

## Checklist

| # | Check | Principle | ASVS | Severity |
|---|-------|-----------|------|----------|
| 1 | Generic errors in production | ERROR_MESSAGES | 16.5.1 | 🟡 MEDIUM |
| 2 | No stack traces to client | ERROR_MESSAGES | 16.5.1 | 🟡 MEDIUM |
| 3 | Third-party errors wrapped | THIRD_PARTY_ERRORS | 16.5.1 | 🟡 MEDIUM |
| 4 | Panic recovery middleware | PANIC_RECOVERY | 16.5.4 | 🟠 HIGH |
| 5 | No sensitive data in logs | LOGGING_SECURITY | 16.3.1 | 🟠 HIGH |
| 6 | Trace ID for correlation | ERROR_MESSAGES | 16.3.1 | 🟢 LOW |
| 7 | Auth attempts logged | SECURITY_LOGGING | 16.3.1 | 🟠 HIGH |
| 8 | Authorization failures logged | SECURITY_LOGGING | 16.3.2 | 🟡 MEDIUM |
| 9 | Logs protected from tampering | LOG_PROTECTION | 16.4.2 | 🟡 MEDIUM |
| 10 | Secrets from vault/env (not hardcoded) | DATA_PROTECTION | 14.1.1 | 🔴 CRITICAL |
| 11 | Approved crypto algorithms | CRYPTOGRAPHY | 11.3.2 | 🟠 HIGH |
| 12 | crypto/rand for random values | CRYPTOGRAPHY | 11.5.1 | 🟠 HIGH |
