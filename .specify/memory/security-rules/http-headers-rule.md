# HTTP Security Headers Rules

Based on Penetration Test Findings:
- 5.7: Missing security headers (Medium)
- 5.10: Cache poisoning risks (Medium)

## Principle SECURITY_HEADERS. HTTP Security Headers

All HTTP responses MUST include appropriate security headers:

### Required Headers

| Header | Value | Purpose |
|--------|-------|---------|
| `Content-Security-Policy` | See below | Prevent XSS, clickjacking, data injection |
| `X-Frame-Options` | `DENY` or `SAMEORIGIN` | Prevent clickjacking |
| `X-Content-Type-Options` | `nosniff` | Prevent MIME sniffing |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Force HTTPS |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Control referrer info |
| `Permissions-Policy` | Feature restrictions | Limit browser features |

**ASVS Requirements**:
- V14.4.3 – Verify that a Content Security Policy (CSP) is in place
- V14.4.4 – Verify that all responses contain X-Content-Type-Options: nosniff
- V14.4.5 – Verify that a Strict-Transport-Security header is included

**Rationale**: Security headers provide defense-in-depth against common web attacks including XSS, clickjacking, and protocol downgrade attacks.

### Go Implementation Example

```go
// ✅ CORRECT: Security headers middleware
func SecurityHeadersMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Prevent clickjacking
        w.Header().Set("X-Frame-Options", "DENY")
        
        // Prevent MIME sniffing
        w.Header().Set("X-Content-Type-Options", "nosniff")
        
        // Force HTTPS (1 year)
        w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        
        // Control referrer
        w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")
        
        // Content Security Policy
        w.Header().Set("Content-Security-Policy", 
            "default-src 'self'; "+
            "script-src 'self'; "+
            "style-src 'self' 'unsafe-inline'; "+
            "img-src 'self' data: https:; "+
            "font-src 'self'; "+
            "frame-ancestors 'none'; "+
            "base-uri 'self'; "+
            "form-action 'self'")
        
        // Permissions Policy
        w.Header().Set("Permissions-Policy", 
            "geolocation=(), microphone=(), camera=()")
        
        next.ServeHTTP(w, r)
    })
}

// ❌ WRONG: No security headers
func badHandler(w http.ResponseWriter, r *http.Request) {
    // Missing all security headers
    w.Write([]byte("Hello"))
}
```

## Principle CSP_POLICY. Content Security Policy

CSP MUST be configured to prevent XSS:

- `default-src 'self'` as baseline
- Avoid `'unsafe-inline'` for scripts (use nonces or hashes)
- Avoid `'unsafe-eval'` (prevents eval(), Function())
- `frame-ancestors 'none'` or `'self'` to prevent clickjacking
- Report violations with `report-uri` or `report-to`

**CSP Directives Reference**:

| Directive | Purpose | Recommended Value |
|-----------|---------|-------------------|
| `default-src` | Default policy | `'self'` |
| `script-src` | JavaScript sources | `'self'` (avoid `'unsafe-inline'`) |
| `style-src` | CSS sources | `'self'` (or `'unsafe-inline'` if needed) |
| `img-src` | Image sources | `'self' data: https:` |
| `font-src` | Font sources | `'self'` |
| `connect-src` | XHR, WebSocket, fetch | `'self'` |
| `frame-ancestors` | Who can embed | `'none'` |
| `base-uri` | Base URL restriction | `'self'` |
| `form-action` | Form submission targets | `'self'` |

**Go Implementation Example**:

```go
// ✅ CORRECT: Strict CSP with nonces for inline scripts
func CSPMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        nonce := generateSecureNonce()
        ctx := context.WithValue(r.Context(), "csp-nonce", nonce)
        
        csp := fmt.Sprintf(
            "default-src 'self'; "+
            "script-src 'self' 'nonce-%s'; "+
            "style-src 'self' 'nonce-%s'; "+
            "img-src 'self' data: https:; "+
            "frame-ancestors 'none'; "+
            "base-uri 'self'; "+
            "form-action 'self'; "+
            "report-uri /csp-report",
            nonce, nonce)
        
        w.Header().Set("Content-Security-Policy", csp)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

## Principle CORS_POLICY. CORS Configuration

Cross-Origin Resource Sharing MUST be configured securely:

- `Access-Control-Allow-Origin` MUST NOT be `*` for authenticated endpoints
- Allowed origins MUST be explicitly listed
- `Access-Control-Allow-Credentials: true` requires specific origin (NOT `*`)
- Preflight requests (`OPTIONS`) MUST be handled correctly
- `Access-Control-Allow-Methods` and `Access-Control-Allow-Headers` MUST be restrictive

**Go Implementation Example**:

```go
// ✅ CORRECT: Restrictive CORS configuration
var allowedOrigins = map[string]bool{
    "https://app.example.com": true,
    "https://admin.example.com": true,
}

func CORSMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        origin := r.Header.Get("Origin")
        
        if allowedOrigins[origin] {
            w.Header().Set("Access-Control-Allow-Origin", origin)
            w.Header().Set("Access-Control-Allow-Credentials", "true")
            w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE")
            w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization, X-CSRF-Token")
            w.Header().Set("Access-Control-Max-Age", "86400")
        }
        
        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusNoContent)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}

// ❌ WRONG: Wildcard CORS with credentials
w.Header().Set("Access-Control-Allow-Origin", "*")
w.Header().Set("Access-Control-Allow-Credentials", "true") // Invalid combination!
```

## Principle CACHE_CONTROL. Cache Control Headers

Sensitive responses MUST have appropriate cache controls:

- Authentication responses: `Cache-Control: no-store`
- User-specific data: `Cache-Control: private, no-cache`
- Static public assets: `Cache-Control: public, max-age=31536000`
- API responses with user data: `Cache-Control: no-store`

**Go Implementation Example**:

```go
// ✅ CORRECT: No caching for sensitive data
func sensitiveDataHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Cache-Control", "no-store")
    w.Header().Set("Pragma", "no-cache")
    // Return sensitive data
}

// ✅ CORRECT: Cache static assets
func staticAssetHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Cache-Control", "public, max-age=31536000, immutable")
    // Serve static file
}
```

## Principle HEADER_INJECTION. Header Injection Prevention

User input MUST NOT be used in HTTP headers without validation:

- Validate and sanitize any user input used in headers
- Remove or encode newline characters (`\r`, `\n`)
- Validate redirect URLs against allowlist
- `Host` and `X-Forwarded-*` headers MUST be validated

**Go Implementation Example**:

```go
// ✅ CORRECT: Validate redirect URL
func safeRedirect(w http.ResponseWriter, r *http.Request, targetURL string) {
    // Only allow relative paths or same-origin
    parsed, err := url.Parse(targetURL)
    if err != nil || (parsed.Host != "" && parsed.Host != r.Host) {
        http.Error(w, "Invalid redirect", http.StatusBadRequest)
        return
    }
    http.Redirect(w, r, targetURL, http.StatusFound)
}

// ❌ WRONG: Unvalidated redirect
func badRedirect(w http.ResponseWriter, r *http.Request) {
    target := r.URL.Query().Get("redirect")
    http.Redirect(w, r, target, http.StatusFound) // Open redirect!
}

// ✅ CORRECT: Sanitize header values
func setCustomHeader(w http.ResponseWriter, value string) {
    // Remove newlines to prevent header injection
    sanitized := strings.ReplaceAll(value, "\r", "")
    sanitized = strings.ReplaceAll(sanitized, "\n", "")
    w.Header().Set("X-Custom-Header", sanitized)
}
```

## AI Agent Requirements

When checking HTTP security headers:

- MUST verify security headers middleware exists
- MUST verify CSP is configured (not just present, but restrictive)
- MUST verify X-Frame-Options is set
- MUST verify X-Content-Type-Options is set
- MUST verify HSTS is configured for production
- MUST verify CORS doesn't use wildcard with credentials
- MUST verify sensitive responses have no-store cache control
- MUST flag missing security headers middleware as HIGH
- MUST flag `Access-Control-Allow-Origin: *` with credentials as CRITICAL

## Security Checklist

- [ ] Security headers middleware applied to all routes
- [ ] CSP configured with restrictive policy
- [ ] X-Frame-Options set to DENY or SAMEORIGIN
- [ ] X-Content-Type-Options set to nosniff
- [ ] HSTS enabled for production
- [ ] Referrer-Policy configured
- [ ] CORS allows specific origins only
- [ ] No wildcard CORS with credentials
- [ ] Sensitive responses not cached
- [ ] User input not used in headers without validation
- [ ] Redirect URLs validated
