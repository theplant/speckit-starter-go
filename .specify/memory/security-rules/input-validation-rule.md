# Input Validation Security Rules

## Principle INPUT_VALIDATION

**Requirements**:
- ALL inputs → validate server-side (client-side is UX only)
- Use ALLOWLIST (what IS allowed) not blocklist
- Validate: type, length, format, range
- Enum fields → check against explicit list

**Detection Pattern**:
```go
// ❌ Using query param directly without validation
lang := r.URL.Query().Get("language")
setLanguageCookie(w, lang) // injection!

// ✅ Validate against allowlist
if !allowedLanguages[lang] { return error }
```

## Principle LENGTH_LIMITS

**Recommended Limits**:
| Field | Max Length |
|-------|------------|
| Username | 100 |
| Email | 255 |
| Password | 72 (bcrypt) |
| Name | 100 |
| Title | 200 |
| Description | 5000 |
| URL | 2048 |

**Detection Pattern**:
```go
// ❌ No length check before DB save
// ✅ utf8.RuneCountInString(input) <= maxLen
// ✅ r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
```

## Principle URL_VALIDATION

**Requirements**:
- Scheme → ONLY `http`, `https`
- Block: `javascript:`, `data:`, `file:`
- SSRF → block localhost, private IPs

**Detection Pattern**:
```go
// ❌ Using URL from user input directly
// ✅ url.Parse() → check Scheme → check !isInternalAddress(host)
```

## Principle RATE_LIMITING

**Requirements**:
- Login → strict (e.g., 5/min per IP)
- Password reset → strict
- API → reasonable per use case

**Detection**: Look for `rate.Limiter` or similar on auth endpoints.

## Principle CONSISTENT_VALIDATION

**Requirements**:
- Same field → same rules everywhere
- Create = Update validation
- Save = Send validation
- Use shared `Validate()` method

## Principle BUSINESS_LOGIC (ASVS 2.3)

**Requirements**:
- Enforce business rules server-side (NOT client-only)
- Validate logical sequences (e.g., payment before shipping)
- Check quantity/price limits
- Prevent race conditions in transactions

**Detection Pattern**:
```go
// ❌ Trust client-side calculated total
// ✅ Recalculate price server-side from DB
```

## Principle ANTI_AUTOMATION (ASVS 2.4)

**Requirements**:
- CAPTCHA or proof-of-work on sensitive actions
- Detect automated/bot traffic patterns
- Rate limit by IP AND by user/session

## Principle ACCESS_CONTROL (ASVS 9.1)

**Requirements**:
- Deny by default → explicitly grant permissions
- Check authorization on EVERY request
- Validate user owns requested resource (IDOR prevention)
- Admin functions → additional auth checks

**Detection Pattern**:
```go
// ❌ /api/users/{id} — no ownership check
// ✅ if resource.OwnerID != currentUser.ID { return 403 }
```

## Principle PRIVILEGE_ESCALATION (ASVS 9.2)

**Requirements**:
- Users cannot elevate own privileges
- Role changes require admin + re-auth
- Horizontal access (other users' data) blocked

## Checklist

| # | Check | Principle | ASVS | Severity |
|---|-------|-----------|------|----------|
| 1 | Server-side validation exists | INPUT_VALIDATION | 2.2.1 | 🟠 HIGH |
| 2 | Allowlist for enum fields | INPUT_VALIDATION | 2.2.1 | 🟡 MEDIUM |
| 3 | String length limits | LENGTH_LIMITS | 2.2.1 | 🟡 MEDIUM |
| 4 | Request body size limited | LENGTH_LIMITS | 2.2.1 | 🟡 MEDIUM |
| 5 | URLs validated (scheme, host) | URL_VALIDATION | 2.2.1 | 🟡 MEDIUM |
| 6 | Rate limiting on login | RATE_LIMITING | 2.4.1 | 🟠 HIGH |
| 7 | Rate limiting on password reset | RATE_LIMITING | 2.4.1 | 🟠 HIGH |
| 8 | Consistent validation | CONSISTENT_VALIDATION | 2.2.1 | 🟡 MEDIUM |
| 9 | Business logic enforced server-side | BUSINESS_LOGIC | 2.3.1 | 🟠 HIGH |
| 10 | Authorization checked every request | ACCESS_CONTROL | 9.1.1 | 🔴 CRITICAL |
| 11 | Resource ownership validated (IDOR) | ACCESS_CONTROL | 9.1.2 | 🔴 CRITICAL |
| 12 | No privilege escalation possible | PRIVILEGE_ESCALATION | 9.2.1 | 🔴 CRITICAL |
