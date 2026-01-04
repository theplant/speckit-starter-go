# XSS & Injection Security Rules

Based on Penetration Test Findings:
- 5.2: Email template stored XSS (High, CVSS 8.6)
- 5.4: Client-side template injection via Vue.js (Medium)

## Principle OUTPUT_ENCODING. Contextual Output Encoding

All user-controlled content MUST be properly encoded before output:

- HTML context: Use HTML entity encoding (`&lt;`, `&gt;`, `&amp;`, `&quot;`, `&#x27;`)
- JavaScript context: Use JavaScript Unicode encoding
- URL context: Use URL encoding (`%XX`)
- CSS context: Use CSS encoding
- JSON context: Ensure proper JSON serialization with escaping
- Content MUST be encoded based on output context (NOT input-time sanitization alone)

**ASVS Requirements**:
- V5.3.4 – Verify that output encoding is consistently applied to all untrusted HTML input
- V5.1.1 – Verify that output encoding is applied consistently to dynamic content

**Rationale**: XSS vulnerabilities allow attackers to execute scripts in victims' browsers, leading to session hijacking, data theft, and privilege escalation.

### Go Implementation Example

```go
import "html/template"

// ✅ CORRECT: Using html/template with automatic escaping
tmpl := template.Must(template.New("page").Parse(`
    <div>Hello, {{.Username}}</div>
`))
tmpl.Execute(w, data) // Automatically escapes HTML

// ✅ CORRECT: Manual HTML escaping
import "html"
safeContent := html.EscapeString(userInput)

// ❌ WRONG: Direct string concatenation
fmt.Fprintf(w, "<div>Hello, %s</div>", userInput) // XSS vulnerable!

// ❌ WRONG: Using text/template for HTML
import "text/template" // NO! Use html/template for HTML output
```

## Principle HTML_SANITIZATION. HTML Content Sanitization

When HTML input is required (e.g., rich text editors, email templates):

- Use allowlist-based HTML sanitization (NOT blocklist)
- Strip dangerous tags: `<script>`, `<iframe>`, `<object>`, `<embed>`, `<form>`, `<input>`
- Strip dangerous attributes: `onclick`, `onerror`, `onload`, `javascript:`, `data:`
- Strip dangerous CSS: `expression()`, `url()` with external resources
- Use established sanitization libraries (e.g., bluemonday for Go)

**Go Implementation Example**:

```go
import "github.com/microcosm-cc/bluemonday"

// ✅ CORRECT: Using bluemonday for HTML sanitization
func sanitizeHTML(input string) string {
    // Strict policy - only allow basic formatting
    p := bluemonday.StrictPolicy()
    return p.Sanitize(input)
}

// ✅ CORRECT: UGC policy for user-generated content
func sanitizeUserContent(input string) string {
    p := bluemonday.UGCPolicy()
    // Remove any remaining dangerous patterns
    p.AllowAttrs("href").OnElements("a")
    p.RequireNoReferrerOnLinks(true)
    return p.Sanitize(input)
}

// ❌ WRONG: Blocklist-based sanitization
func badSanitize(input string) string {
    // Easily bypassed with variations like <ScRiPt>, <script/>, etc.
    return strings.ReplaceAll(input, "<script>", "")
}
```

## Principle TEMPLATE_INJECTION. Template Injection Prevention

Server-side and client-side templates MUST be protected from injection:

- User input MUST NOT be used in template directives
- Template expressions MUST NOT evaluate user-controlled data
- Vue.js: Avoid `v-html` with user content; use `{{ }}` with caution for double-encoding
- Go templates: Never use `template.HTML()` with unsanitized user input

**ASVS Requirements**:
- V5.2.4 – Verify that the application protects against template injection attacks

**Go Implementation Example**:

```go
// ❌ WRONG: Template injection vulnerability
userTemplate := r.FormValue("template")
tmpl, _ := template.New("user").Parse(userTemplate) // CRITICAL: User controls template!
tmpl.Execute(w, data)

// ✅ CORRECT: Fixed templates, user data as values only
tmpl := template.Must(template.ParseFiles("templates/page.html"))
tmpl.Execute(w, map[string]string{
    "UserContent": sanitizeHTML(userInput),
})

// ❌ WRONG: Marking unsanitized content as safe
content := template.HTML(userInput) // NEVER do this with user input!

// ✅ CORRECT: Sanitize before marking as safe HTML
content := template.HTML(bluemonday.UGCPolicy().Sanitize(userInput))
```

## Principle SQL_INJECTION. SQL Injection Prevention

Database queries MUST use parameterized queries:

- ALWAYS use parameterized queries or prepared statements
- NEVER concatenate user input into SQL strings
- ORM methods MUST use proper parameter binding
- Raw SQL MUST use placeholders (`$1`, `?`, or named parameters)

**Go Implementation Example**:

```go
// ✅ CORRECT: Parameterized query with GORM
db.Where("email = ?", userEmail).First(&user)

// ✅ CORRECT: Parameterized raw SQL
db.Raw("SELECT * FROM users WHERE email = $1", userEmail).Scan(&user)

// ❌ WRONG: String concatenation
db.Raw("SELECT * FROM users WHERE email = '" + userEmail + "'").Scan(&user) // SQL INJECTION!

// ❌ WRONG: fmt.Sprintf for SQL
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", userEmail) // SQL INJECTION!
```

## Principle COMMAND_INJECTION. Command Injection Prevention

System commands MUST NOT include user input:

- Avoid executing system commands with user input
- If unavoidable, use strict allowlist validation
- Use parameterized command execution (separate command and arguments)
- NEVER use shell interpolation with user input

**Go Implementation Example**:

```go
// ✅ CORRECT: Separate command and arguments
cmd := exec.Command("convert", inputFile, "-resize", "100x100", outputFile)

// ❌ WRONG: Shell command with user input
cmd := exec.Command("sh", "-c", "convert " + userFilename + " output.jpg") // INJECTION!
```

## AI Agent Requirements

When checking XSS and injection security:

- MUST verify all user output uses appropriate encoding
- MUST verify HTML templates use `html/template` (NOT `text/template`)
- MUST verify HTML sanitization uses allowlist approach (e.g., bluemonday)
- MUST verify SQL queries use parameterized queries
- MUST flag any `template.HTML()` usage with user input as CRITICAL
- MUST flag any string concatenation in SQL as CRITICAL
- MUST flag any `v-html` usage with user content as HIGH
- MUST verify no user input in template directives

## Security Checklist

- [ ] All HTML output uses `html/template` or manual escaping
- [ ] Rich text fields use bluemonday or equivalent sanitization
- [ ] No `template.HTML()` with unsanitized user input
- [ ] All SQL uses parameterized queries
- [ ] No string concatenation in database queries
- [ ] Vue.js templates avoid `v-html` with user content
- [ ] Server-side templates don't evaluate user input
- [ ] Command execution doesn't include user input
