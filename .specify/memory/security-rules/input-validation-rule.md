# Input Validation Security Rules

Based on Penetration Test Findings:
- 5.5: Language parameter cookie injection (Medium)
- 5.6: Missing rate limiting (Medium)
- 5.8: Excessively long string inputs (Medium)
- 5.9: Phishing via unrestricted URL embedding (Medium)
- 5.15: Inconsistent validation between save and send (Low)

## Principle INPUT_VALIDATION. Server-Side Input Validation

All user input MUST be validated on the server side:

- Client-side validation is for UX only; server MUST re-validate
- Use allowlist validation (define what IS allowed) over blocklist (define what ISN'T)
- Validate type, length, format, and range
- Validation MUST be consistent across all entry points
- Reject invalid input early with clear error messages

**ASVS Requirements**:
- V5.1.1 – Verify that the application has defenses against HTTP parameter pollution
- V5.1.3 – Verify that all input is validated using positive validation (allowlists)

**Rationale**: Client-side validation can be bypassed. Server-side validation is the only reliable defense against malicious input.

### Go Implementation Example

```go
// ✅ CORRECT: Comprehensive input validation
type CreateUserRequest struct {
    Email    string `json:"email" validate:"required,email,max=255"`
    Name     string `json:"name" validate:"required,min=1,max=100"`
    Age      int    `json:"age" validate:"gte=0,lte=150"`
    Language string `json:"language" validate:"oneof=en ja zh"`
}

func (r *CreateUserRequest) Validate() error {
    validate := validator.New()
    if err := validate.Struct(r); err != nil {
        return fmt.Errorf("validation failed: %w", err)
    }
    return nil
}

// ✅ CORRECT: Allowlist validation for enum-like fields
var allowedLanguages = map[string]bool{
    "en": true, "ja": true, "zh": true,
}

func validateLanguage(lang string) error {
    if !allowedLanguages[lang] {
        return fmt.Errorf("invalid language: must be one of en, ja, zh")
    }
    return nil
}

// ❌ WRONG: No server-side validation
func badHandler(w http.ResponseWriter, r *http.Request) {
    lang := r.URL.Query().Get("language")
    // Using language directly without validation
    setLanguageCookie(w, lang) // Cookie injection risk!
}
```

## Principle LENGTH_LIMITS. String Length Limits

All string inputs MUST have maximum length limits:

- Define maximum lengths appropriate for each field
- Enforce limits at database schema level AND application level
- Consider UTF-8 multi-byte characters (1 character ≠ 1 byte)
- Reject oversized inputs early to prevent DoS

**Field Length Guidelines**:

| Field Type | Recommended Max Length |
|------------|----------------------|
| Username | 50 |
| Email | 255 |
| Password | 72 (bcrypt limit) |
| Name | 100 |
| Title | 200 |
| Description | 5000 |
| URL | 2048 |
| Phone | 20 |

**Go Implementation Example**:

```go
// ✅ CORRECT: Length validation
func validateInput(input string, maxLen int, fieldName string) error {
    // Check byte length (for storage)
    if len(input) > maxLen*4 { // Assume max 4 bytes per UTF-8 char
        return fmt.Errorf("%s exceeds maximum byte length", fieldName)
    }
    
    // Check character length
    if utf8.RuneCountInString(input) > maxLen {
        return fmt.Errorf("%s must be at most %d characters", fieldName, maxLen)
    }
    
    return nil
}

// ✅ CORRECT: Request body size limit
func limitedBodyHandler(w http.ResponseWriter, r *http.Request) {
    r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // 1MB limit
    // Parse body...
}

// ❌ WRONG: No length limits
func badHandler(w http.ResponseWriter, r *http.Request) {
    title := r.FormValue("title")
    // Saving very long title directly - UI/DB issues!
    db.Create(&Article{Title: title})
}
```

## Principle URL_VALIDATION. URL Validation

User-provided URLs MUST be validated:

- Validate URL format and scheme (allowlist: `http`, `https`)
- Block dangerous schemes: `javascript:`, `data:`, `file:`, `vbscript:`
- Validate domain against allowlist for sensitive operations
- Prevent SSRF by blocking internal IPs and localhost
- Sanitize URLs before embedding in HTML/templates

**Go Implementation Example**:

```go
// ✅ CORRECT: URL validation
func validateURL(rawURL string) error {
    parsed, err := url.Parse(rawURL)
    if err != nil {
        return fmt.Errorf("invalid URL format")
    }
    
    // Only allow http/https
    if parsed.Scheme != "http" && parsed.Scheme != "https" {
        return fmt.Errorf("URL must use http or https scheme")
    }
    
    // Block internal addresses (SSRF prevention)
    host := parsed.Hostname()
    if isInternalAddress(host) {
        return fmt.Errorf("internal addresses not allowed")
    }
    
    return nil
}

func isInternalAddress(host string) bool {
    // Block localhost, private IPs, etc.
    if host == "localhost" || host == "127.0.0.1" || host == "::1" {
        return true
    }
    ip := net.ParseIP(host)
    if ip != nil {
        return ip.IsLoopback() || ip.IsPrivate() || ip.IsLinkLocalUnicast()
    }
    return false
}

// ❌ WRONG: No URL validation
func badHandler(w http.ResponseWriter, r *http.Request) {
    url := r.FormValue("callback_url")
    // Using URL in notifications without validation - phishing risk!
    sendNotification(fmt.Sprintf("Click here: %s", url))
}
```

## Principle RATE_LIMITING. Rate Limiting

Sensitive endpoints MUST implement rate limiting:

- Login/authentication: Strict limits (e.g., 5 attempts per minute)
- Password reset: Strict limits
- API endpoints: Reasonable limits based on use case
- Admin operations: Additional limits
- Rate limits MUST be per-user/IP, not global

**Go Implementation Example**:

```go
import "golang.org/x/time/rate"

// ✅ CORRECT: Per-IP rate limiting
type RateLimiter struct {
    limiters map[string]*rate.Limiter
    mu       sync.RWMutex
    rate     rate.Limit
    burst    int
}

func NewRateLimiter(r rate.Limit, b int) *RateLimiter {
    return &RateLimiter{
        limiters: make(map[string]*rate.Limiter),
        rate:     r,
        burst:    b,
    }
}

func (rl *RateLimiter) GetLimiter(ip string) *rate.Limiter {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    if limiter, exists := rl.limiters[ip]; exists {
        return limiter
    }
    
    limiter := rate.NewLimiter(rl.rate, rl.burst)
    rl.limiters[ip] = limiter
    return limiter
}

func RateLimitMiddleware(rl *RateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ip := getClientIP(r)
            limiter := rl.GetLimiter(ip)
            
            if !limiter.Allow() {
                http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}

// Usage: 5 requests per second, burst of 10
loginLimiter := NewRateLimiter(5, 10)
http.Handle("/login", RateLimitMiddleware(loginLimiter)(loginHandler))

// ❌ WRONG: No rate limiting on login
http.HandleFunc("/login", loginHandler) // Brute force vulnerable!
```

## Principle CONSISTENT_VALIDATION. Consistent Validation

Validation MUST be consistent across all operations:

- Same field MUST have same validation rules everywhere
- Validation on create MUST match validation on update
- Validation on save MUST match validation on send/execute
- Use shared validation functions to ensure consistency

**Go Implementation Example**:

```go
// ✅ CORRECT: Shared validation
type Message struct {
    Content string `json:"content"`
    AltText string `json:"alt_text"`
}

// Single source of truth for validation rules
func (m *Message) Validate() error {
    if len(m.Content) > 5000 {
        return fmt.Errorf("content must be at most 5000 characters")
    }
    if len(m.AltText) > 1500 { // LINE API limit
        return fmt.Errorf("alt_text must be at most 1500 characters")
    }
    return nil
}

// Used consistently in both create and send
func createMessage(m *Message) error {
    if err := m.Validate(); err != nil {
        return err
    }
    return db.Create(m)
}

func sendMessage(m *Message) error {
    if err := m.Validate(); err != nil {
        return err
    }
    return lineAPI.Send(m)
}

// ❌ WRONG: Inconsistent validation
func badCreate(m *Message) error {
    // No validation - message saved
    return db.Create(m)
}

func badSend(m *Message) error {
    // LINE API rejects - inconsistent behavior!
    return lineAPI.Send(m)
}
```

## AI Agent Requirements

When checking input validation:

- MUST verify all user inputs have server-side validation
- MUST verify string length limits exist for all text fields
- MUST verify URLs are validated before use
- MUST verify enum-like fields use allowlist validation
- MUST verify rate limiting exists on login/auth endpoints
- MUST verify validation is consistent across operations
- MUST flag missing server-side validation as HIGH
- MUST flag missing rate limiting on auth as HIGH
- MUST flag unvalidated URLs as MEDIUM

## Security Checklist

- [ ] All inputs validated server-side
- [ ] Allowlist validation used where applicable
- [ ] String length limits enforced
- [ ] Request body size limited
- [ ] URLs validated before use
- [ ] Dangerous URL schemes blocked
- [ ] Rate limiting on authentication
- [ ] Rate limiting on sensitive operations
- [ ] Validation consistent across create/update
- [ ] Validation consistent across save/send
- [ ] Error messages don't leak sensitive info
