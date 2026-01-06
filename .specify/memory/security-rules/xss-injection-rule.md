# XSS & Injection Security Rules

## Principle OUTPUT_ENCODING

**Requirements**:
- HTML output → use `html/template` (NOT `text/template`)
- Direct output → use `html.EscapeString()`
- NEVER use `fmt.Fprintf(w, "<div>%s</div>", userInput)`

**Detection Pattern**:
```go
// ❌ text/template, fmt.Fprintf with user input
// ✅ html/template auto-escapes: {{.UserInput}}
```

## Principle HTML_SANITIZATION

**Requirements**:
- Rich text → use allowlist sanitizer (bluemonday)
- NEVER use blocklist (`strings.ReplaceAll(input, "<script>", "")`)

**Dangerous Elements** (must strip):
`<script>`, `<iframe>`, `<object>`, `<embed>`, `<form>`, `onclick`, `onerror`, `javascript:`

**Detection Pattern**:
```go
// ❌ template.HTML(userInput) — marks unsafe content as safe
// ✅ template.HTML(bluemonday.UGCPolicy().Sanitize(userInput))
```

## Principle TEMPLATE_INJECTION

**Requirements**:
- User input → ONLY as template values, NEVER in template code
- Go: NEVER `template.Parse(userInput)`
- Vue.js: AVOID `v-html` with user content

**Detection Pattern**:
```go
// ❌ template.New("x").Parse(r.FormValue("tpl")) — user controls template!
// ✅ template.ParseFiles("fixed.html") — fixed template
```

## Principle SQL_INJECTION

**Requirements**:
- ALWAYS use parameterized queries: `?` or `$1`
- NEVER concatenate: `"WHERE id = '" + id + "'"`
- NEVER use fmt.Sprintf for SQL

**Detection Pattern**:
```go
// ❌ db.Raw("...'" + userInput + "'...")
// ❌ fmt.Sprintf("SELECT ... '%s'", userInput)
// ✅ db.Where("email = ?", userInput)
// ✅ db.Raw("SELECT ... WHERE id = $1", userInput)
```

## Principle COMMAND_INJECTION

**Requirements**:
- AVOID exec with user input
- If needed → separate command and arguments
- NEVER use shell interpolation

**Detection Pattern**:
```go
// ❌ exec.Command("sh", "-c", "cmd " + userInput)
// ✅ exec.Command("cmd", arg1, arg2) — no shell
```

## Principle LDAP_INJECTION (ASVS 1.2)

**Requirements**:
- LDAP queries → use parameterized/escaped input
- NEVER concatenate user input into LDAP filters

**Detection Pattern**:
```go
// ❌ fmt.Sprintf("(uid=%s)", userInput)
// ✅ ldap.EscapeFilter(userInput) or parameterized
```

## Principle XPATH_INJECTION (ASVS 1.2)

**Requirements**:
- XPath queries → parameterize or escape input
- NEVER build XPath from user strings

## Principle DESERIALIZATION (ASVS 1.5)

**Requirements**:
- Untrusted data → validate before deserialize
- Avoid native serialization (gob) with untrusted input
- JSON preferred over binary formats
- Whitelist allowed types if polymorphic

**Detection Pattern**:
```go
// ❌ gob.Decode(untrustedInput, &obj)
// ✅ json.Unmarshal with struct (typed)
```

## Checklist

| # | Check | Principle | ASVS | Severity |
|---|-------|-----------|------|----------|
| 1 | HTML uses `html/template` | OUTPUT_ENCODING | 1.2.1 | 🔴 CRITICAL |
| 2 | No `template.HTML(userInput)` | HTML_SANITIZATION | 1.2.1 | 🔴 CRITICAL |
| 3 | SQL uses parameterized queries | SQL_INJECTION | 1.2.1 | 🔴 CRITICAL |
| 4 | No string concat in SQL | SQL_INJECTION | 1.2.1 | 🔴 CRITICAL |
| 5 | Rich text uses bluemonday | HTML_SANITIZATION | 1.3.1 | 🟠 HIGH |
| 6 | No `v-html` with user content | TEMPLATE_INJECTION | 1.2.1 | 🟠 HIGH |
| 7 | No user input in template code | TEMPLATE_INJECTION | 1.2.1 | 🔴 CRITICAL |
| 8 | No shell interpolation | COMMAND_INJECTION | 1.2.1 | 🔴 CRITICAL |
| 9 | LDAP queries parameterized | LDAP_INJECTION | 1.2.1 | 🔴 CRITICAL |
| 10 | Safe deserialization (typed JSON) | DESERIALIZATION | 1.5.1 | 🟠 HIGH |
