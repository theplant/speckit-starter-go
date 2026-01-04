# File Upload Security Rules

Based on Penetration Test Finding 5.3: Unrestricted File Uploads (High)

## Principle FILE_TYPE_VALIDATION. File Type Validation

All file uploads MUST validate file types on the server side:

- File extension MUST be checked against an allowlist (NOT blocklist)
- MIME type MUST be validated via Content-Type header AND magic bytes
- Magic bytes (file signature) MUST be verified to prevent extension spoofing
- Double extensions (e.g., `file.php.jpg`) MUST be rejected
- Null byte attacks (e.g., `file.php%00.jpg`) MUST be prevented

**ASVS Requirements**:
- V12.1.1 – Verify that the application validates uploaded files to ensure they match expected file type

**Rationale**: Unrestricted file uploads can lead to remote code execution, malware distribution, XSS via SVG/HTML files, and server resource exhaustion.

### File Signature (Magic Bytes) Reference

| File Type | Extension | Magic Bytes (Hex) |
|-----------|-----------|-------------------|
| JPEG | .jpg, .jpeg | `FF D8 FF` |
| PNG | .png | `89 50 4E 47 0D 0A 1A 0A` |
| GIF | .gif | `47 49 46 38` |
| PDF | .pdf | `25 50 44 46` |
| ZIP | .zip | `50 4B 03 04` |

### Go Implementation Example

```go
// ✅ CORRECT: Comprehensive file type validation
func validateFileType(file multipart.File, filename string, allowedTypes map[string][]byte) error {
    // 1. Check extension against allowlist
    ext := strings.ToLower(filepath.Ext(filename))
    magicBytes, allowed := allowedTypes[ext]
    if !allowed {
        return fmt.Errorf("file extension %s not allowed", ext)
    }
    
    // 2. Check for double extensions
    if strings.Count(filename, ".") > 1 {
        return fmt.Errorf("double extensions not allowed")
    }
    
    // 3. Read and verify magic bytes
    header := make([]byte, len(magicBytes))
    if _, err := file.Read(header); err != nil {
        return fmt.Errorf("failed to read file header: %w", err)
    }
    file.Seek(0, 0) // Reset file pointer
    
    if !bytes.HasPrefix(header, magicBytes) {
        return fmt.Errorf("file content does not match extension")
    }
    
    return nil
}

var allowedImageTypes = map[string][]byte{
    ".jpg":  {0xFF, 0xD8, 0xFF},
    ".jpeg": {0xFF, 0xD8, 0xFF},
    ".png":  {0x89, 0x50, 0x4E, 0x47},
    ".gif":  {0x47, 0x49, 0x46, 0x38},
}

// ❌ WRONG: Only checking extension
func badValidation(filename string) bool {
    ext := filepath.Ext(filename)
    return ext == ".jpg" || ext == ".png" // Easily bypassed!
}

// ❌ WRONG: Only checking Content-Type header
func badMimeCheck(contentType string) bool {
    return contentType == "image/jpeg" // Client can spoof this!
}
```

## Principle FILE_SIZE_LIMIT. File Size Limits

File uploads MUST enforce size limits:

- Maximum file size MUST be enforced on both client AND server
- Server-side size limit MUST NOT rely on client-side validation alone
- Size limits MUST be appropriate for the file type
- Request body size MUST be limited to prevent DoS

**Go Implementation Example**:

```go
// ✅ CORRECT: Server-side size limit
func uploadHandler(w http.ResponseWriter, r *http.Request) {
    // Limit request body size (e.g., 10MB)
    r.Body = http.MaxBytesReader(w, r.Body, 10<<20)
    
    err := r.ParseMultipartForm(10 << 20) // 10MB max memory
    if err != nil {
        http.Error(w, "File too large", http.StatusRequestEntityTooLarge)
        return
    }
    
    file, header, err := r.FormFile("upload")
    if err != nil {
        http.Error(w, "Invalid file", http.StatusBadRequest)
        return
    }
    defer file.Close()
    
    // Additional size check
    if header.Size > 5<<20 { // 5MB limit for images
        http.Error(w, "File exceeds 5MB limit", http.StatusRequestEntityTooLarge)
        return
    }
}

// ❌ WRONG: No size limit
func badUploadHandler(w http.ResponseWriter, r *http.Request) {
    r.ParseMultipartForm(0) // No limit - DoS vulnerability!
}
```

## Principle FILE_STORAGE. Secure File Storage

Uploaded files MUST be stored securely:

- Files MUST be stored outside the web root
- Original filenames MUST NOT be used for storage (use generated UUIDs)
- File metadata (original name, uploader, timestamp) MUST be stored in database
- Executable permissions MUST NOT be set on uploaded files
- Files SHOULD be served via a separate domain or with proper Content-Disposition

**Go Implementation Example**:

```go
// ✅ CORRECT: Secure file storage
func storeFile(file multipart.File, originalName string) (string, error) {
    // Generate random filename
    newFilename := uuid.New().String() + filepath.Ext(originalName)
    
    // Store outside web root
    storagePath := filepath.Join("/var/app/uploads", newFilename)
    
    dst, err := os.OpenFile(storagePath, os.O_WRONLY|os.O_CREATE, 0644)
    if err != nil {
        return "", err
    }
    defer dst.Close()
    
    if _, err := io.Copy(dst, file); err != nil {
        return "", err
    }
    
    // Store metadata in database
    saveFileMetadata(newFilename, originalName, time.Now())
    
    return newFilename, nil
}

// ❌ WRONG: Using original filename
func badStoreFile(file multipart.File, originalName string) error {
    dst, _ := os.Create(filepath.Join("./public/uploads", originalName)) // Path traversal risk!
    io.Copy(dst, file)
    return nil
}
```

## Principle FILE_SERVING. Secure File Serving

Files MUST be served with appropriate security headers:

- `Content-Type` MUST be set correctly (NOT inferred from extension)
- `Content-Disposition: attachment` for downloads to prevent inline execution
- `X-Content-Type-Options: nosniff` to prevent MIME sniffing
- Sensitive files MUST require authentication

**Go Implementation Example**:

```go
// ✅ CORRECT: Secure file serving
func serveFile(w http.ResponseWriter, r *http.Request, filePath string) {
    // Set security headers
    w.Header().Set("X-Content-Type-Options", "nosniff")
    w.Header().Set("Content-Disposition", "attachment; filename=\"download\"")
    w.Header().Set("Content-Type", "application/octet-stream")
    
    http.ServeFile(w, r, filePath)
}

// For images that need to be displayed inline:
func serveImage(w http.ResponseWriter, r *http.Request, filePath string) {
    // Verify it's actually an image
    contentType := detectContentType(filePath)
    if !strings.HasPrefix(contentType, "image/") {
        http.Error(w, "Not an image", http.StatusBadRequest)
        return
    }
    
    w.Header().Set("X-Content-Type-Options", "nosniff")
    w.Header().Set("Content-Type", contentType)
    
    http.ServeFile(w, r, filePath)
}
```

## Principle DANGEROUS_FILES. Dangerous File Type Blocking

Certain file types MUST always be blocked:

- Executable files: `.exe`, `.dll`, `.so`, `.bat`, `.cmd`, `.sh`, `.ps1`
- Server-side scripts: `.php`, `.asp`, `.aspx`, `.jsp`, `.cgi`
- HTML/Script files: `.html`, `.htm`, `.svg` (contains JavaScript), `.xml`
- Archive bombs: Check decompressed size before extraction

**Go Implementation Example**:

```go
var blockedExtensions = map[string]bool{
    ".exe": true, ".dll": true, ".so": true,
    ".bat": true, ".cmd": true, ".sh": true, ".ps1": true,
    ".php": true, ".asp": true, ".aspx": true, ".jsp": true,
    ".html": true, ".htm": true, ".svg": true,
}

func isBlockedExtension(filename string) bool {
    ext := strings.ToLower(filepath.Ext(filename))
    return blockedExtensions[ext]
}
```

## AI Agent Requirements

When checking file upload security:

- MUST verify server-side file type validation exists
- MUST verify magic bytes validation (not just extension)
- MUST verify file size limits are enforced server-side
- MUST verify files are stored outside web root
- MUST verify original filenames are not used for storage
- MUST verify dangerous file types are blocked
- MUST flag any upload without type validation as HIGH
- MUST flag any upload without size limit as MEDIUM

## Security Checklist

- [ ] File extension validated against allowlist
- [ ] Magic bytes verified for all uploads
- [ ] MIME type validated (not trusted from client)
- [ ] Double extensions rejected
- [ ] File size limited on server side
- [ ] Request body size limited
- [ ] Files stored outside web root
- [ ] Random filenames used for storage
- [ ] Dangerous file types blocked
- [ ] Files served with security headers
- [ ] Content-Disposition set for downloads
