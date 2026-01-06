# File Upload Security Rules

## Principle FILE_TYPE_VALIDATION

**Requirements**:
- Extension → check against ALLOWLIST (not blocklist)
- Magic bytes → verify file signature matches extension
- Double extensions → reject (e.g., `file.php.jpg`)
- Content-Type header → don't trust, client can spoof

**Magic Bytes Reference**:
| Type | Magic Bytes |
|------|-------------|
| JPEG | `FF D8 FF` |
| PNG | `89 50 4E 47` |
| GIF | `47 49 46 38` |
| PDF | `25 50 44 46` |

**Detection Pattern**:
```go
// ❌ Only checking extension or Content-Type
// ✅ Check extension + read first bytes + verify magic signature
```

## Principle FILE_SIZE_LIMIT

**Requirements**:
- Server-side size limit → use `http.MaxBytesReader()`
- Request body limit → prevent DoS
- Per-file limit → appropriate for file type

**Detection Pattern**:
```go
// ❌ r.ParseMultipartForm(0) — no limit
// ✅ r.Body = http.MaxBytesReader(w, r.Body, 10<<20) — 10MB limit
```

## Principle FILE_STORAGE

**Requirements**:
- Storage location → OUTSIDE web root
- Filename → generate UUID, NEVER use original
- Permissions → 0644 (no execute)
- Metadata → store original name in database

**Detection Pattern**:
```go
// ❌ os.Create("./public/uploads/" + originalName) — path traversal!
// ✅ uuid.New().String() + ext, store in /var/app/uploads/
```

## Principle DANGEROUS_FILES

**Always Block**:
- Executables: `.exe`, `.dll`, `.so`, `.bat`, `.sh`, `.ps1`
- Scripts: `.php`, `.asp`, `.aspx`, `.jsp`, `.cgi`
- HTML/JS: `.html`, `.htm`, `.svg`, `.xml`

## Principle FILE_SERVING

**Requirements**:
- Downloads → `Content-Disposition: attachment`
- All files → `X-Content-Type-Options: nosniff`
- Content-Type → detect from content, not extension

## Principle ARCHIVE_SECURITY (ASVS 5.2.3)

**Requirements**:
- Check uncompressed size before extracting
- Limit max files in archive
- Detect zip bombs (compression ratio check)
- Block symlinks in archives (unless explicitly needed)

**Detection Pattern**:
```go
// ❌ Extract without size check
zip.ExtractAll(archive, dest)

// ✅ Check uncompressed size first
if totalUncompressedSize > maxAllowed { reject() }
```

## Principle PATH_TRAVERSAL (ASVS 5.3.2)

**Requirements**:
- Sanitize user-provided filenames
- Block `..`, absolute paths, null bytes
- Use generated filenames internally
- Validate final path is within allowed directory

**Detection Pattern**:
```go
// ❌ filepath.Join(uploadDir, userFilename)
// ✅ Clean path + verify prefix
cleanPath := filepath.Clean(userFilename)
if strings.Contains(cleanPath, "..") { reject() }
fullPath := filepath.Join(uploadDir, cleanPath)
if !strings.HasPrefix(fullPath, uploadDir) { reject() }
```

## Principle ANTIVIRUS_SCAN (ASVS 5.4.3)

**Requirements**:
- Scan uploaded files with antivirus (L2+)
- Quarantine suspicious files
- Block known malicious signatures

## Checklist

| # | Check | Principle | ASVS | Severity |
|---|-------|-----------|------|----------|
| 1 | Extension allowlist validation | FILE_TYPE_VALIDATION | 5.2.2 | 🟠 HIGH |
| 2 | Magic bytes verification | FILE_TYPE_VALIDATION | 5.2.2 | 🟠 HIGH |
| 3 | Server-side size limit | FILE_SIZE_LIMIT | 5.2.1 | 🟡 MEDIUM |
| 4 | Files stored outside web root | FILE_STORAGE | 5.3.1 | 🟠 HIGH |
| 5 | Random filenames (not original) | FILE_STORAGE | 5.3.2 | 🟠 HIGH |
| 6 | Dangerous extensions blocked | DANGEROUS_FILES | 5.2.2 | 🟠 HIGH |
| 7 | Security headers on file serve | FILE_SERVING | 5.4.1 | 🟡 MEDIUM |
| 8 | Archive size checked before extract | ARCHIVE_SECURITY | 5.2.3 | 🟠 HIGH |
| 9 | Path traversal blocked | PATH_TRAVERSAL | 5.3.2 | 🔴 CRITICAL |
| 10 | Symlinks blocked in archives | ARCHIVE_SECURITY | 5.2.5 | 🟡 MEDIUM |
| 11 | Antivirus scan (L2+) | ANTIVIRUS_SCAN | 5.4.3 | 🟡 MEDIUM |
