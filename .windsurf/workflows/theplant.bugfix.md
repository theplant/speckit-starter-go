---
description: Execute Bug Fix Flow with reproduction-first debugging following constitution principles.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Fix bugs using reproduction-first debugging following constitution principles. When a bug is reported (curl request/response, error logs, user report), follow this systematic workflow.

## Execution Steps

### Phase 1: Capture & Analyze

1. **Capture the failing request**
   - Save exact curl command, request body, headers, and error response
   
2. **Identify the endpoint**
   - Extract HTTP method, path, and path parameters
   
3. **Extract test data**
   - Parse JSON request body to create test fixtures
   
4. **Document expected vs actual**
   - Note what response should be vs what was returned

### Phase 2: Reproduce with Integration Test

1. **Write a failing test FIRST** (Principle INTEGRATION_TESTING: NO mocking)

2. **Test through root mux ServeHTTP** (Principle SERVEHTTP_TESTING)

3. **Use table-driven design** (Principle TABLE_DRIVEN) with descriptive name:
   ```go
   {name: "BUG-123: PUT workflow returns INVALID_REQUEST for valid branch step"}
   ```

4. **Use generated structs** (Principle SCHEMA_FIRST) - NO `map[string]interface{}`

5. **Setup realistic fixtures** via GORM representing bug's preconditions

6. **Verify test fails** with same error as reported bug

### Phase 3: Root Cause Analysis (Principle ROOT_CAUSE)

1. **Trace backward**: Follow call chain from error response to origin

2. **Distinguish symptoms from causes**: Error message is symptom, find root cause

3. **Use debugger/logging**: Add temporary logging to understand control flow

4. **Form hypotheses**: List potential causes and systematically eliminate them

5. **Document findings**: Record root cause before implementing fix

### Phase 4: Fix & Verify

1. **Fix at the source**: Address root cause, NOT symptoms

2. **Run the reproduction test**:
// turbo
```bash
go test -v -run "BUG-" ./...
```

3. **Run full test suite**:
// turbo
```bash
go test -v -race ./...
```

4. **Update documentation**: If bug revealed unclear API behavior, update docs

## Test Naming Convention

- Format: `"BUG-<ID>: <brief description of the bug>"`
- Example: `"BUG-123: PUT workflow returns INVALID_REQUEST for valid branch step"`
- Include bug ID in test case struct for traceability

## AI Agent Requirements

- MUST write failing reproduction test BEFORE attempting any fix
- MUST NOT skip reproduction step even for "obvious" bugs
- MUST verify test fails with reported error before proceeding
- MUST apply Root Cause Tracing (Principle ROOT_CAUSE) - no superficial fixes
- MUST run full test suite after fix to catch regressions (Principle CONTINUOUS_TESTING)

## Debugging Process

1. **Reproduce**: Create reliable reproduction case
2. **Observe**: Gather evidence through logs, debugger, tests
3. **Hypothesize**: Form theories about root cause
4. **Test**: Design experiments to validate/invalidate hypotheses
5. **Fix**: Implement fix addressing root cause
6. **Verify**: Ensure fix works and doesn't break existing functionality
7. **Document**: Update docs/tests to prevent regression
