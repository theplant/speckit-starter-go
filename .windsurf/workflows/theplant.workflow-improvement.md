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


## Execution Steps

First, create a task plan using `update_plan`:

```
Call update_plan with:
- explanation: "Workflow improvement from conversation learnings"
- plan: [
    {"step": "Identify issues and blockers from conversation", "status": "pending"},
    {"step": "Categorize issues by target workflow", "status": "pending"},
    {"step": "Read target workflow files", "status": "pending"},
    {"step": "Apply updates to workflow files", "status": "pending"},
    {"step": "Update update_plan Execution Steps if workflow steps changed", "status": "pending"},
    {"step": "Sync with constitution and templates", "status": "pending"}
  ]
```

Then execute each step, updating status to `in_progress` before starting and `completed` after finishing.

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

7. **CRITICAL: Sync with Constitution** - When updating ANY workflow, you MUST also update:
   - `.specify/memory/constitution.md` - The project constitution (bump version if needed)
   - `.specify/templates/plan-template.md` - Implementation plan template
   - `.specify/templates/tasks-template.md` - Task list template
   - Keep all documents consistent with the same patterns and principles

8. **CRITICAL: Update `update_plan` Execution Steps** - When modifying a workflow's steps, you MUST also update the `update_plan` call in the "Execution Steps" section to match:
   - The plan steps in the `update_plan` call MUST reflect the actual workflow steps
   - Step names should be concise but descriptive
   - All workflows should have an `update_plan` call at the start of "Execution Steps"

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
|------------|------------------|
| API/Type mismatches, ogen patterns | `theplant.openapi.md` |
| Error handling, ErrorHandler, ErrorCodeMapping | `theplant.openapi.md` |
| Selector strategies, test patterns | `theplant.e2e-testing.md` |
| Handler patterns, data persistence | `theplant.msw-mock-backend.md` |
| Investigation strategies, debugging | `theplant.root-cause-tracing.md` |
| Data flow tracing | `theplant.system-exploration.md` |
| Test data patterns | `theplant.test-data-seeding.md` |
| Integration testing, ServeHTTP | `theplant.integration-test.md` |
| HTTP routes, builder pattern | `theplant.routes.md` |
| Test database, testcontainers | `theplant.testdb.md` |

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

**CRITICAL: Also update constitution and templates:**
1. Read `.specify/memory/constitution.md` to find related sections
2. Update constitution with the same patterns (bump version if significant change)
3. Update `.specify/templates/plan-template.md` if it affects project structure or testing strategy
4. Update `.specify/templates/tasks-template.md` if it affects task organization or code structure

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


