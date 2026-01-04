# CSRF & Session Security Rules

Based on Penetration Test Finding 5.1: CSRF-Based Forced Login Vulnerability (Critical, CVSS 9.1)

## Principle CSRF_PROTECTION. Cross-Site Request Forgery Protection

All state-changing endpoints MUST implement CSRF protection:

- State-changing requests (POST, PUT, DELETE, PATCH) MUST validate CSRF tokens
- CSRF tokens MUST be cryptographically random, single-use, and tied to user session
- Login endpoints MUST implement CSRF protection (prevent forced login attacks)
- CSRF tokens MUST be transmitted via custom headers or hidden form fields (NOT URL parameters)
- Server MUST reject requests with missing or invalid CSRF tokens with HTTP 403

**ASVS Requirements**:
- V4.3.1 – Verify that a CSRF protection mechanism is in place for all state-changing authenticated requests
- V3.2.1 – Verify that session identifiers are not accepted from requests from different origins

**Rationale**: CSRF attacks force authenticated users to perform unintended actions. Without CSRF protection, attackers can construct malicious pages that cause victims' browsers to submit forged requests.

### Go Implementation Example

```go
// ✅ CORRECT: CSRF middleware with token validation
func CSRFMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Method == "POST" || r.Method == "PUT" || r.Method == "DELETE" || r.Method == "PATCH" {
            token := r.Header.Get("X-CSRF-Token")
            sessionToken := getSessionCSRFToken(r)
            if token == "" || !secureCompare(token, sessionToken) {
                http.Error(w, "CSRF token validation failed", http.StatusForbidden)
                return
            }
        }
        next.ServeHTTP(w, r)
    })
}

// ❌ WRONG: No CSRF protection on login
func LoginHandler(w http.ResponseWriter, r *http.Request) {
    // Directly processing login without CSRF check
    username := r.FormValue("username")
    password := r.FormValue("password")
    // ... authenticate
}
```

## Principle ORIGIN_VALIDATION. Origin and Referer Header Validation

Sensitive endpoints MUST validate request origin:

- Server MUST check `Origin` header matches allowed origins
- Server MUST check `Referer` header for same-origin requests
- Cross-origin requests MUST be rejected for sensitive operations
- Allowed origins MUST be explicitly configured (NO wildcards for sensitive endpoints)

**Go Implementation Example**:

```go
// ✅ CORRECT: Origin validation
func ValidateOrigin(r *http.Request, allowedOrigins []string) bool {
    origin := r.Header.Get("Origin")
    if origin == "" {
        origin = r.Header.Get("Referer")
    }
    for _, allowed := range allowedOrigins {
        if strings.HasPrefix(origin, allowed) {
            return true
        }
    }
    return false
}
```

## Principle COOKIE_SECURITY. Secure Cookie Configuration

Authentication cookies MUST be configured securely:

- Cookies MUST set `SameSite=Strict` (or `Lax` with additional CSRF protection)
- Cookies MUST set `Secure` flag (HTTPS only)
- Cookies MUST set `HttpOnly` flag (prevent JavaScript access)
- Cookies MUST set appropriate `Path` and `Domain` scope
- Session cookies MUST have reasonable expiration times

**ASVS Requirements**:
- V3.4.3 – Verify that authentication cookies are set with the SameSite=Strict attribute
- V3.4.1 – Verify that cookie-based session tokens have the Secure attribute set
- V3.4.2 – Verify that cookie-based session tokens have the HttpOnly attribute set

**Go Implementation Example**:

```go
// ✅ CORRECT: Secure cookie settings
http.SetCookie(w, &http.Cookie{
    Name:     "session",
    Value:    sessionToken,
    Path:     "/",
    HttpOnly: true,
    Secure:   true,
    SameSite: http.SameSiteStrictMode,
    MaxAge:   3600, // 1 hour
})

// ❌ WRONG: Insecure cookie settings
http.SetCookie(w, &http.Cookie{
    Name:     "session",
    Value:    sessionToken,
    SameSite: http.SameSiteLaxMode, // Allows cross-site POST with cookies
})
```

## Principle SESSION_MANAGEMENT. Secure Session Management

Session management MUST follow security best practices:

- Session IDs MUST be cryptographically random (minimum 128 bits entropy)
- Session IDs MUST be regenerated after authentication
- Sessions MUST be invalidated on logout (server-side)
- Concurrent session limits SHOULD be enforced
- Session timeout MUST be implemented (idle and absolute)

**Go Implementation Example**:

```go
// ✅ CORRECT: Regenerate session after login
func LoginHandler(w http.ResponseWriter, r *http.Request) {
    // ... authenticate user
    
    // Invalidate old session
    oldSession := getSession(r)
    if oldSession != nil {
        invalidateSession(oldSession.ID)
    }
    
    // Create new session
    newSessionID := generateSecureToken(32) // 256 bits
    createSession(newSessionID, userID)
    setSessionCookie(w, newSessionID)
}
```

## AI Agent Requirements

When checking CSRF and session security:

- MUST verify all POST/PUT/DELETE/PATCH endpoints have CSRF token validation
- MUST verify login endpoint has CSRF protection
- MUST check cookie settings for Secure, HttpOnly, SameSite flags
- MUST verify Origin/Referer validation exists for sensitive operations
- MUST check session regeneration after authentication
- MUST verify session invalidation on logout
- MUST NOT approve code that uses `SameSite=None` without explicit justification
- MUST flag any state-changing endpoint without CSRF protection as CRITICAL

## Security Checklist

- [ ] All state-changing endpoints validate CSRF tokens
- [ ] Login endpoint has CSRF protection
- [ ] Cookies use `SameSite=Strict` or `Lax` with CSRF tokens
- [ ] Cookies have `Secure` and `HttpOnly` flags
- [ ] Origin/Referer headers are validated
- [ ] Sessions are regenerated after login
- [ ] Sessions are invalidated on logout
- [ ] Session tokens are cryptographically random
