# Error Handling Security Rules

Based on Penetration Test Findings:
- 5.11: Verbose error messages exposing internal details (Medium)
- 5.14: Backend error details leaked to client (Low)
- 5.15: Raw third-party API errors exposed (Low)

## Principle ERROR_MESSAGES. Secure Error Messages

Error responses MUST NOT expose sensitive internal details:

- Production errors MUST return generic user-friendly messages
- Internal details (stack traces, SQL queries, file paths) MUST NOT be exposed
- Error codes SHOULD be used for client-side handling
- Detailed errors MUST be logged server-side for debugging
- Development mode MAY show detailed errors (controlled by environment)

**ASVS Requirements**:
- V7.4.1 – Verify that a generic message is shown when an unexpected error occurs
- V7.4.2 – Verify that exception handling is used across the codebase

**Rationale**: Detailed error messages help attackers understand system internals, identify vulnerabilities, and craft targeted attacks.

### Go Implementation Example

```go
// ✅ CORRECT: Generic error response with internal logging
type ErrorResponse struct {
    Code    string `json:"code"`
    Message string `json:"message"`
    TraceID string `json:"trace_id,omitempty"` // For support correlation
}

func handleError(w http.ResponseWriter, r *http.Request, err error, statusCode int) {
    traceID := getTraceID(r.Context())
    
    // Log detailed error internally
    log.WithFields(log.Fields{
        "trace_id": traceID,
        "error":    err.Error(),
        "path":     r.URL.Path,
        "method":   r.Method,
    }).Error("Request failed")
    
    // Return generic message to client
    resp := ErrorResponse{
        Code:    getErrorCode(err),
        Message: getPublicMessage(err),
        TraceID: traceID,
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    json.NewEncoder(w).Encode(resp)
}

func getPublicMessage(err error) string {
    // Map internal errors to public messages
    switch {
    case errors.Is(err, ErrNotFound):
        return "The requested resource was not found"
    case errors.Is(err, ErrUnauthorized):
        return "Authentication required"
    case errors.Is(err, ErrForbidden):
        return "You don't have permission to perform this action"
    case errors.Is(err, ErrValidation):
        return "The provided data is invalid"
    default:
        return "An unexpected error occurred" // Generic fallback
    }
}

// ❌ WRONG: Exposing internal error details
func badHandler(w http.ResponseWriter, r *http.Request) {
    result, err := db.Query("SELECT * FROM users WHERE id = ?", id)
    if err != nil {
        // Exposes SQL query and database details!
        http.Error(w, fmt.Sprintf("Database error: %v", err), 500)
        return
    }
}
```

## Principle ERROR_CODES. Standardized Error Codes

Use standardized error codes for consistent error handling:

- Define application-specific error codes
- Map errors to appropriate HTTP status codes
- Document error codes for API consumers
- Use error codes for client-side handling logic

**Error Code Structure Example**:

```go
// ✅ CORRECT: Standardized error codes
type AppError struct {
    Code       string
    Message    string
    HTTPStatus int
    Internal   error
}

var (
    ErrNotFound = &AppError{
        Code:       "NOT_FOUND",
        Message:    "Resource not found",
        HTTPStatus: 404,
    }
    ErrValidation = &AppError{
        Code:       "VALIDATION_ERROR",
        Message:    "Invalid input data",
        HTTPStatus: 400,
    }
    ErrUnauthorized = &AppError{
        Code:       "UNAUTHORIZED",
        Message:    "Authentication required",
        HTTPStatus: 401,
    }
    ErrForbidden = &AppError{
        Code:       "FORBIDDEN",
        Message:    "Permission denied",
        HTTPStatus: 403,
    }
    ErrRateLimit = &AppError{
        Code:       "RATE_LIMIT_EXCEEDED",
        Message:    "Too many requests",
        HTTPStatus: 429,
    }
    ErrInternal = &AppError{
        Code:       "INTERNAL_ERROR",
        Message:    "An unexpected error occurred",
        HTTPStatus: 500,
    }
)
```

## Principle THIRD_PARTY_ERRORS. Third-Party Error Handling

Errors from external services MUST be wrapped:

- Never expose raw third-party API errors to clients
- Log original error for debugging
- Return generic error with correlation ID
- Map third-party errors to application error codes

**Go Implementation Example**:

```go
// ✅ CORRECT: Wrapping third-party errors
func sendLINEMessage(msg *Message) error {
    resp, err := lineClient.Send(msg)
    if err != nil {
        // Log the detailed error
        log.WithError(err).Error("LINE API call failed")
        
        // Return wrapped error without exposing LINE internals
        return fmt.Errorf("failed to send message: %w", ErrExternalService)
    }
    
    if resp.StatusCode != 200 {
        // Log the detailed response
        log.WithFields(log.Fields{
            "status_code": resp.StatusCode,
            "body":        resp.Body,
        }).Error("LINE API returned error")
        
        // Return generic error
        return ErrMessageDeliveryFailed
    }
    
    return nil
}

// ❌ WRONG: Exposing third-party error details
func badSendMessage(msg *Message) error {
    resp, err := lineClient.Send(msg)
    if err != nil {
        // Exposes LINE API internals!
        return fmt.Errorf("LINE error: %v, response: %s", err, resp.Body)
    }
    return nil
}
```

## Principle ENVIRONMENT_ERRORS. Environment-Aware Error Details

Error detail level SHOULD vary by environment:

- Production: Minimal details, generic messages
- Staging: May include error types for debugging
- Development: Full details for debugging

**Go Implementation Example**:

```go
// ✅ CORRECT: Environment-aware error handling
var hideErrorDetails = os.Getenv("HIDE_ERROR_DETAILS") == "true"

func formatErrorResponse(err error) map[string]interface{} {
    resp := map[string]interface{}{
        "error": map[string]interface{}{
            "code":    getErrorCode(err),
            "message": getPublicMessage(err),
        },
    }
    
    // Only include details in development
    if !hideErrorDetails {
        resp["error"].(map[string]interface{})["details"] = err.Error()
    }
    
    return resp
}

// Environment configuration:
// Production: HIDE_ERROR_DETAILS=true
// Development: HIDE_ERROR_DETAILS=false
```

## Principle PANIC_RECOVERY. Panic Recovery

Application MUST recover from panics gracefully:

- Use recovery middleware to catch panics
- Log panic details with stack trace
- Return generic 500 error to client
- Prevent service crash from single request

**Go Implementation Example**:

```go
// ✅ CORRECT: Panic recovery middleware
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                // Log panic with stack trace
                log.WithFields(log.Fields{
                    "panic":  err,
                    "stack":  string(debug.Stack()),
                    "path":   r.URL.Path,
                    "method": r.Method,
                }).Error("Panic recovered")
                
                // Return generic error
                w.Header().Set("Content-Type", "application/json")
                w.WriteHeader(http.StatusInternalServerError)
                json.NewEncoder(w).Encode(ErrorResponse{
                    Code:    "INTERNAL_ERROR",
                    Message: "An unexpected error occurred",
                })
            }
        }()
        
        next.ServeHTTP(w, r)
    })
}

// ❌ WRONG: No panic recovery
func badServer() {
    http.ListenAndServe(":8080", handler) // Panic crashes entire server!
}
```

## Principle LOGGING_SECURITY. Secure Error Logging

Error logs MUST NOT contain sensitive data:

- Never log passwords, tokens, or API keys
- Mask or truncate sensitive fields
- Use structured logging for easier filtering
- Set appropriate log retention policies

**Go Implementation Example**:

```go
// ✅ CORRECT: Secure logging
func logRequest(r *http.Request, err error) {
    // Mask sensitive headers
    headers := make(map[string]string)
    for k, v := range r.Header {
        if k == "Authorization" || k == "Cookie" {
            headers[k] = "[REDACTED]"
        } else {
            headers[k] = strings.Join(v, ", ")
        }
    }
    
    log.WithFields(log.Fields{
        "path":    r.URL.Path,
        "method":  r.Method,
        "headers": headers,
        "error":   err.Error(),
    }).Error("Request failed")
}

// ❌ WRONG: Logging sensitive data
func badLog(r *http.Request, password string) {
    log.Printf("Login failed for user with password: %s", password) // NEVER!
}
```

## AI Agent Requirements

When checking error handling:

- MUST verify errors don't expose internal details in production
- MUST verify third-party errors are wrapped
- MUST verify panic recovery middleware exists
- MUST verify error logging doesn't include sensitive data
- MUST verify environment-aware error detail levels
- MUST flag raw error exposure as MEDIUM
- MUST flag third-party error exposure as MEDIUM
- MUST flag sensitive data in logs as HIGH

## Security Checklist

- [ ] Generic error messages in production
- [ ] Detailed errors logged server-side
- [ ] Error codes defined and documented
- [ ] Third-party errors wrapped
- [ ] Panic recovery middleware in place
- [ ] Environment-aware error details
- [ ] Sensitive data masked in logs
- [ ] Stack traces not exposed to clients
- [ ] Correlation IDs for error tracking
- [ ] HTTP status codes match error types
