# CSRF & Session Security Rules

## Principle CSRF_PROTECTION

**Requirements**:
- POST/PUT/DELETE/PATCH → MUST validate CSRF token
- Login endpoint → MUST have CSRF protection
- Token → cryptographically random, single-use, tied to session
- Transmission → via header `X-CSRF-Token` or hidden field (NOT URL)
- Missing/invalid token → HTTP 403

**Detection Pattern**:
```go
// ❌ WRONG: No CSRF check on state-changing handler
func Handler(w http.ResponseWriter, r *http.Request) {
    // Direct processing without token validation
}

// ✅ CORRECT: Validate token from header, compare with session
r.Header.Get("X-CSRF-Token") // Must match session token
```

## Principle COOKIE_SECURITY

**Required Flags**:
| Flag | Value | Reason |
|------|-------|--------|
| `SameSite` | `Strict` (best) or `Lax` | Prevent cross-site requests |
| `Secure` | `true` | HTTPS only |
| `HttpOnly` | `true` | Block JS access |

**Detection Pattern**:
```go
// ❌ Missing: SameSite, Secure, HttpOnly not set
http.SetCookie(w, &http.Cookie{Name: "session", Value: token})

// ✅ All flags set
http.SetCookie(w, &http.Cookie{..., HttpOnly: true, Secure: true, SameSite: http.SameSiteStrictMode})
```

## Principle SESSION_MANAGEMENT

**Requirements**:
- Session ID → minimum 128 bits entropy
- After login → regenerate session ID
- On logout → invalidate server-side
- Timeout → implement idle and absolute

**Detection Pattern**:
```go
// ❌ WRONG: Reusing old session after login
// ✅ CORRECT: invalidateSession(old) → createSession(new)
```

## Principle PASSWORD_SECURITY (ASVS 6.2)

**Requirements**:
- Minimum length → 8 chars (recommend 15+)
- Maximum → at least 64 chars allowed
- NO composition rules (uppercase/special NOT required)
- Check against breached password list (top 3000+)
- Allow paste and password managers
- NO periodic rotation (change only if compromised)
- Hash with Argon2id/bcrypt/scrypt (NOT MD5/SHA256 alone)

**Detection Pattern**:
```go
// ❌ Weak: short min, composition rules, plain hash
// ✅ len >= 8, breached check, bcrypt.GenerateFromPassword(..., 12)
```

## Principle CREDENTIAL_ERRORS (ASVS 6.3.8)

**Requirements**:
- Generic error → "Invalid credentials" (NOT "user not found")
- Consistent response time (prevent timing attacks)
- Same behavior for login/register/password-reset

## Principle SESSION_TIMEOUT (ASVS 7.2)

**Requirements**:
- Idle timeout → 15-30 min for sensitive apps
- Absolute timeout → max 12-24 hours
- Re-auth for sensitive operations

## Checklist

| # | Check | Principle | ASVS | Severity |
|---|-------|-----------|------|----------|
| 1 | State-changing endpoints have CSRF token | CSRF_PROTECTION | 3.5.2 | 🔴 CRITICAL |
| 2 | Login has CSRF protection | CSRF_PROTECTION | 3.5.2 | 🔴 CRITICAL |
| 3 | Cookie: SameSite=Strict or Lax | COOKIE_SECURITY | 3.3.1 | 🟠 HIGH |
| 4 | Cookie: Secure=true | COOKIE_SECURITY | 3.3.1 | 🟠 HIGH |
| 5 | Cookie: HttpOnly=true | COOKIE_SECURITY | 3.3.1 | 🟠 HIGH |
| 6 | Session regenerated after login | SESSION_MANAGEMENT | 7.1.1 | 🟠 HIGH |
| 7 | Session invalidated on logout | SESSION_MANAGEMENT | 7.1.3 | 🟡 MEDIUM |
| 8 | Password min 8 chars, no composition rules | PASSWORD_SECURITY | 6.2.1 | 🟠 HIGH |
| 9 | Password checked against breached list | PASSWORD_SECURITY | 6.2.12 | 🟠 HIGH |
| 10 | Passwords hashed with Argon2id/bcrypt | PASSWORD_SECURITY | 11.4.2 | 🔴 CRITICAL |
| 11 | Generic auth error messages | CREDENTIAL_ERRORS | 6.3.8 | 🟡 MEDIUM |
| 12 | Session idle/absolute timeout | SESSION_TIMEOUT | 7.2.1 | 🟡 MEDIUM |
