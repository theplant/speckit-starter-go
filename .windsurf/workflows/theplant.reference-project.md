---
description: Download reference projects using tiged for pattern and best practice reference
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Download a reference project repository into the current workspace for pattern and best practice reference, ensuring it's excluded from version control.

## Execution Steps

First, create a task plan using `update_plan`:

```
Call update_plan with:
- explanation: "Download reference project <project-name>"
- plan: [
    {"step": "Update .gitignore to exclude .reference/", "status": "pending"},
    {"step": "Download reference project with tiged", "status": "pending"},
    {"step": "Verify download and git status", "status": "pending"},
    {"step": "Document reference in .reference/README.md", "status": "pending"}
  ]
```

Then execute each step, updating status to `in_progress` before starting and `completed` after finishing.

## Steps

### Step 1: Critical Rules

**IMPORTANT: Read and follow ALL rules before proceeding.**

1. **Reference projects are read-only** - NEVER modify files in the reference project directory
2. **Always use `--mode=git`** - This ensures proper git history download with tiged
3. **Always update `.gitignore`** - Reference projects MUST be excluded from version control
4. **Use `.reference/` directory** - All reference projects go in `.reference/<project-name>/`
5. **Document the source** - Always note the source repository URL

### Step 2: Update .gitignore

Add the `.reference/` directory to `.gitignore` if not already present:

```bash
# Check if .reference/ is already in .gitignore
grep -q "^\.reference/" .gitignore 2>/dev/null || echo ".reference/" >> .gitignore
```

**Why**: Reference projects should never be committed to version control. They are external dependencies for learning/reference only.

### Step 3: Download Reference Project

Use `npx tiged` to download the reference project:

```bash
# Create .reference directory if it doesn't exist
mkdir -p .reference

# Download the reference project
# Replace <github-org/repo> with the actual repository
# Replace <project-name> with a short descriptive name
npx tiged --mode=git <github-org/repo> .reference/<project-name>
```

**Example** (shadcn-admin-go):
```bash
mkdir -p .reference
npx tiged --mode=git theplant/shadcn-admin-go .reference/shadcn-admin-go
```

**Why `--mode=git`**: This mode uses git clone internally, which is more reliable for private repos and preserves more metadata.

### Step 4: Verify Download

Confirm the reference project was downloaded correctly:

```bash
# List the downloaded project structure
ls -la .reference/<project-name>/

# Verify .gitignore excludes it
git status --porcelain .reference/
```

**Expected**: The `.reference/` directory should not appear in git status (ignored).

### Step 5: Document Reference

Create or update `.reference/README.md` to document available reference projects:

```markdown
# Reference Projects

These projects are downloaded for pattern and best practice reference only.
They are excluded from version control via `.gitignore`.

## Available References

### <project-name>
- **Source**: https://github.com/<org>/<repo>
- **Downloaded**: <date>
- **Purpose**: <brief description of what patterns to reference>

## Usage

To re-download a reference project:
```bash
npx tiged --mode=git <org>/<repo> .reference/<project-name>
```
```

## Common Reference Projects

| Project | Command | Purpose |
|---------|---------|---------|
| shadcn-admin-go | `npx tiged --mode=git theplant/shadcn-admin-go .reference/shadcn-admin-go` | Go admin panel patterns with shadcn UI |
