# HTTP Security Headers Rules

## Principle SECURITY_HEADERS

**Required Headers**:
| Header | Value |
|--------|-------|
| `X-Frame-Options` | `DENY` or `SAMEORIGIN` |
| `X-Content-Type-Options` | `nosniff` |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Content-Security-Policy` | See CSP below |

**Detection**: Look for middleware that sets these headers on all responses.

## Principle CSP_POLICY

**Recommended CSP**:
```
default-src 'self'; script-src 'self'; style-src 'self'; 
img-src 'self' data: https:; frame-ancestors 'none'; 
base-uri 'self'; form-action 'self'
```

**Rules**:
- AVOID `'unsafe-inline'` for scripts → use nonces
- AVOID `'unsafe-eval'` → blocks eval()
- `frame-ancestors 'none'` → prevents clickjacking

## Principle CORS_POLICY

**Requirements**:
- `Access-Control-Allow-Origin` → explicit origins, NOT `*` with auth
- `Access-Control-Allow-Credentials: true` → REQUIRES specific origin
- INVALID: `Allow-Origin: *` + `Allow-Credentials: true`

**Detection Pattern**:
```go
// ❌ w.Header().Set("Access-Control-Allow-Origin", "*")
//    w.Header().Set("Access-Control-Allow-Credentials", "true")
// ✅ Check origin against allowlist, set specific origin
```

## Principle CACHE_CONTROL

**Rules**:
| Content Type | Cache-Control |
|--------------|---------------|
| Auth responses | `no-store` |
| User data | `private, no-cache` |
| Static assets | `public, max-age=31536000` |

## Principle HEADER_INJECTION

**Requirements**:
- Redirect URLs → validate against allowlist
- User input in headers → strip `\r\n`
- Open redirect = HIGH severity

**Detection Pattern**:
```go
// ❌ http.Redirect(w, r, r.URL.Query().Get("redirect"), 302)
// ✅ Validate parsed.Host == "" or == r.Host before redirect
```

## Principle TLS_SECURITY (ASVS 15.1)

**Requirements**:
- TLS 1.2+ only (disable TLS 1.0, 1.1, SSL)
- Strong cipher suites (AES-GCM, ChaCha20)
- Valid certificates (not expired, correct domain)
- HSTS with long max-age (31536000)

## Principle EXTERNAL_RESOURCES (ASVS 3.6)

**Requirements**:
- Subresource Integrity (SRI) for external scripts/styles
- CDN resources → include `integrity` attribute
- Verify external content hasn't been tampered

**Detection Pattern**:
```html
<!-- ❌ No integrity check -->
<script src="https://cdn.example.com/lib.js"></script>

<!-- ✅ With SRI -->
<script src="https://cdn.example.com/lib.js" 
  integrity="sha384-..." crossorigin="anonymous"></script>
```

## Principle API_SECURITY (ASVS 4.1)

**Requirements**:
- API versioning in URL or header
- Consistent error response format
- Rate limiting per endpoint
- Request size limits

## Principle JWT_SECURITY (ASVS 8.1)

**Requirements**:
- Validate signature with correct algorithm
- Check `exp`, `iat`, `nbf` claims
- Verify `iss` and `aud` claims
- NEVER use `alg: none`
- Secret key → min 256 bits

**Detection Pattern**:
```go
// ❌ Trusting alg from token header
// ✅ Explicitly set expected algorithm in verification
jwt.Parse(token, keyFunc, jwt.WithValidMethods([]string{"HS256"}))
```

## Checklist

| # | Check | Principle | ASVS | Severity |
|---|-------|-----------|------|----------|
| 1 | Security headers middleware exists | SECURITY_HEADERS | 3.4.1 | 🟠 HIGH |
| 2 | CSP configured (restrictive) | CSP_POLICY | 3.4.1 | 🟠 HIGH |
| 3 | X-Frame-Options set | SECURITY_HEADERS | 3.4.1 | 🟡 MEDIUM |
| 4 | X-Content-Type-Options: nosniff | SECURITY_HEADERS | 3.4.1 | 🟡 MEDIUM |
| 5 | HSTS enabled (production) | SECURITY_HEADERS | 3.4.1 | 🟡 MEDIUM |
| 6 | CORS: no wildcard + credentials | CORS_POLICY | 3.5.2 | 🔴 CRITICAL |
| 7 | Sensitive data: Cache-Control no-store | CACHE_CONTROL | 3.4.1 | 🟡 MEDIUM |
| 8 | Redirect URLs validated | HEADER_INJECTION | 3.4.1 | 🟠 HIGH |
| 9 | TLS 1.2+ only | TLS_SECURITY | 15.1.1 | 🟠 HIGH |
| 10 | External resources have SRI | EXTERNAL_RESOURCES | 3.6.1 | 🟡 MEDIUM |
| 11 | JWT algorithm explicitly validated | JWT_SECURITY | 8.1.1 | 🔴 CRITICAL |
| 12 | JWT claims verified (exp, iss, aud) | JWT_SECURITY | 8.1.2 | 🟠 HIGH |
