---
description: Apply root cause tracing debugging discipline - trace problems backward through call chain, fix source not symptoms, never give up on complex issues.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Apply systematic root cause analysis before implementing fixes. This ensures problems are solved at their source, preventing recurrence.

## Rationale

Superficial fixes create technical debt and hide underlying architectural problems. Root cause analysis transforms debugging from firefighting into systematic problem-solving that improves overall code quality.

## Core Principles

- Problems MUST be traced backward through the call chain to find the original trigger
- Symptoms MUST be distinguished from root causes
- Fixes MUST address the source of the problem, NOT work around symptoms
- Test cases MUST NOT be removed or weakened to make tests pass
- Debuggers and logging MUST be used to understand control flow
- Multiple potential causes MUST be systematically eliminated
- Root cause MUST be verified through testing before closing the issue

### No-Give-Up Rule (NON-NEGOTIABLE)

AI agents MUST NEVER abandon a problem by:
- Reverting to "simpler" approaches that avoid the actual issue
- Saying "this won't work easily with this architecture"
- Removing tests or features because they're "too complex"
- Giving up after initial failures without exhausting root cause analysis

Instead, AI agents MUST:
1. Continue investigating until the root cause is found
2. Try multiple hypotheses systematically
3. Read source code to understand the actual implementation
4. Document findings even if the fix requires architectural changes
5. Only escalate to the user when genuinely blocked after thorough investigation

## Execution Steps

First, create a task plan using `update_plan`:

```
Call update_plan with:
- explanation: "Root cause tracing for <issue description>"
- plan: [
    {"step": "Create reliable reproduction case", "status": "pending"},
    {"step": "Gather evidence through logs and debugger", "status": "pending"},
    {"step": "Form and rank hypotheses", "status": "pending"},
    {"step": "Test hypotheses systematically", "status": "pending"},
    {"step": "Implement fix addressing root cause", "status": "pending"},
    {"step": "Verify fix with reproduction test", "status": "pending"},
    {"step": "Run full test suite", "status": "pending"},
    {"step": "Document findings", "status": "pending"}
  ]
```

Then execute each step, updating status to `in_progress` before starting and `completed` after finishing.

## Debugging Process

### Step 1: Reproduce

Create a reliable reproduction case:
- Write a failing integration test (Principle INTEGRATION_TESTING)
- Use table-driven design with descriptive name (Principle TABLE_DRIVEN)
- Test through root mux ServeHTTP (Principle SERVEHTTP_TESTING)

```go
{name: "BUG-123: PUT product returns 500 for valid request"}
```

### Step 2: Observe

Gather evidence through logs, debugger, tests:
- Add temporary logging in handler and service layers
- Check database state before/after operations
- Examine error responses and stack traces
- Use `t.Logf()` to output intermediate values

```go
// Temporary debug logging
t.Logf("Request body: %s", string(reqBody))
t.Logf("Response: status=%d body=%s", rec.Code, rec.Body.String())
t.Logf("DB state: %+v", product)
```

### Step 3: Hypothesize

Form theories about root cause:
- List all possible causes
- Rank by likelihood based on evidence
- Consider recent changes that might have introduced the issue

Common Go backend root causes:
- **Validation**: Missing or incorrect validation in service layer
- **Database**: Wrong GORM query, missing preload, constraint violation
- **Serialization**: protojson vs json.Marshal mismatch, field tags
- **Context**: Missing `WithContext(ctx)`, context cancellation
- **Concurrency**: Race condition, missing mutex

### Step 4: Test

Design experiments to validate/invalidate hypotheses:
- Test one hypothesis at a time
- Use binary search to narrow down the problem
- Add temporary debug code to verify assumptions

```go
// Test hypothesis: Is the product being created at all?
var count int64
db.Model(&models.Product{}).Count(&count)
t.Logf("Product count after create: %d", count)

// Test hypothesis: Is the error from GORM or service validation?
err := db.Create(&product).Error
t.Logf("Direct GORM error: %v", err)
```

### Step 5: Fix

Implement fix addressing root cause:
- Fix the source, not the symptom
- Ensure fix doesn't break existing functionality
- Keep fix minimal and focused

### Step 6: Verify

Run the reproduction test:
// turbo
```bash
go test -v -run "BUG-" ./...
```

Run full test suite:
// turbo
```bash
go test -v -race ./...
```

### Step 7: Document

Update docs/tests to prevent regression:
- Keep the regression test with bug reference in name
- Update API documentation if behavior was unclear
- Add comments explaining non-obvious fixes

## Example of Violation

```
❌ "The service validation isn't working. Let me just add validation 
    in the handler instead to work around it."
```

This is a violation because it:
- Gives up without finding root cause in service layer
- Adds validation in wrong layer (handlers should be thin)
- Doesn't investigate why service validation failed

## Example of Correct Behavior

```
✅ "The service validation isn't working. Let me trace the root cause:
    1. Reproduce: Write test for missing name validation
    2. Observe: Service.Create() is called but no error returned
    3. Hypothesize: Maybe validation check is wrong? Maybe error not wrapped?
    4. Test: Found `if req.Name != ""` instead of `if req.Name == ""`
    5. Fix: Correct the condition in service layer
    6. Verify: Test passes, run full suite
    7. Document: Added comment explaining validation logic"
```

## Common Error Patterns

### "record not found" but data exists
- Check if using correct ID field (`id` vs `ID` vs `Id`)
- Check if soft delete is filtering results (add `.Unscoped()` to debug)
- Check if query conditions are correct

### "duplicate key" on create
- Check unique constraints in database
- Check if ID is being auto-generated or needs to be set
- Check if using `Create` vs `Save` correctly

### "context canceled" unexpectedly
- Check if handler is using `r.Context()` correctly
- Check if long-running operations check `ctx.Done()`
- Check if client is disconnecting early

### "invalid request" but request looks valid
- Check protojson vs json.Marshal (camelCase vs snake_case)
- Check if required fields are being sent
- Check Content-Type header

### Test passes locally but fails in CI
- Check for race conditions (`go test -race`)
- Check for time-dependent tests (use fixed timestamps)
- Check for order-dependent tests (table truncation)

## Test Fix Discipline

When tests fail:

- NEVER use `t.Skip()` to avoid fixing a complicated test
- ALWAYS trace the root cause of test failures before implementing fixes
- When API changes break tests, update test assertions to match new behavior
- When behavior changes, rewrite test assertions to validate new expected behavior
- Remove tests that test removed functionality

## AI Agent Requirements

- MUST write a failing reproduction test BEFORE attempting any fix
- MUST NOT skip the reproduction step even for "obvious" bugs
- MUST verify the test fails with the reported error before proceeding
- MUST apply Root Cause Tracing - no superficial fixes
- MUST run full test suite after fix to catch regressions (Principle CONTINUOUS_TESTING)
- MUST document the root cause analysis process

## See Also

- `/theplant.bugfix` - Bug fix workflow with reproduction-first debugging
- `/theplant.integration-test` - Integration testing workflow
- `/theplant.system-exploration` - Trace code paths before writing tests
- `/theplant.errors` - Error handling strategy
