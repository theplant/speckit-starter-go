---
description: Summarize conversation issues and blockers, then improve existing ThePlant workflows or create new ones with generalized learnings to prevent future problems.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Analyze conversation issues and update workflows with generalized learnings.

## How to Execute This Workflow

**⚠️ MANDATORY: AI MUST run the workflow-runner command below and follow its output.**

**DO NOT read the Steps section below and execute them manually.** The runner handles:
- Step sequencing and state tracking
- Nested workflow execution
- AI task delegation with proper context

```bash
deno run -A https://raw.githubusercontent.com/theplant/workflow-runner/HEAD/run.ts theplant.workflow-improvement
```

Run this command, then follow the runner's instructions. The runner will tell you what to do next.


## Steps

### Step 1: Rules and Issue Identification

**IMPORTANT: Read and follow ALL rules in this step before proceeding.**

#### Critical Rules for Workflow Updates

1. **ALL rules MUST be in Step 1** - When creating or updating workflows, put all critical rules at the beginning of Step 1. Text before the Steps section is NOT read when workflows are nested-executed.

2. **Use generic names** - Use `MyEntity`, `myField` NOT project-specific names like `User`, `Task`.

3. **Use conventional paths** - Use `src/mocks/handlers.ts` NOT absolute paths like `/Users/john/projects/...`.

4. **Focus on principles** - Document the underlying principle, not the specific instance.

5. **Include problem AND solution** - Always show both the problem pattern and the solution.

6. **Avoid duplication** - Check existing workflows before adding new content.

#### Issue Identification

**Why:** Systematic issue identification ensures no learnings are lost. Each issue becomes a potential workflow improvement.

Analyze the current conversation and document:

```markdown
## Issues Identified

### Issue 1: [Brief description]
- **What happened**: [Description]
- **Root cause**: [Why it happened]
- **Solution applied**: [How it was fixed]
- **Generalized learning**: [What to document]
- **Target workflow**: [Which theplant.*.md to update]

### Issue 2: ...
```

**Why:** Categorization ensures learnings go to the right workflow file, avoiding duplication and keeping workflows focused.

| Issue Type | Target Workflow |
|------------|-----------------|
| API/Type mismatches, Orval hooks | `theplant.openapi-first.md` |
| Selector strategies, test patterns | `theplant.e2e-testing.md` |
| Handler patterns, data persistence | `theplant.msw-mock-backend.md` |
| Investigation strategies, debugging | `theplant.root-cause-tracing.md` |
| Data flow tracing | `theplant.system-exploration.md` |
| Test data patterns | `theplant.test-data-seeding.md` |

### Step 2: Read Target Workflow Files

**Why:** Reading existing content prevents duplication and helps identify the right section for new learnings.

```bash
# Read the workflow file to update
cat .windsurf/workflows/theplant.[target].md
```

Check:
1. Current content structure
2. Appropriate section to add learning
3. Whether similar guidance already exists (avoid duplication)

### Step 3: Apply Updates to Workflow Files

**Why:** Applying updates immediately captures learnings while context is fresh.

Edit the target workflow files with the generalized learnings.

If Needed, Create New Workflow

If learning represents a new discipline, create `theplant.[discipline-name].md`:

```markdown
---
description: [One-line description]
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

[What this workflow achieves]

## Steps

### Step 1: [First deterministic file change]

[Specific file to create/modify with exact content]

### Step 2: [Next file change]

...

### Step N: [Verification]

[How to verify the workflow worked]
```

**New Workflow Criteria:**
- Doesn't fit naturally into existing workflows
- Addresses recurring pattern across projects
- Has enough depth for its own file
- Follows naming: `theplant.[discipline-name].md`


