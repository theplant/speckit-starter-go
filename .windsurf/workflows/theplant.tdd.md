---
description: Execute Test-First Development (TDD) workflow for Go microservices following constitution principles.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Implement new features using Test-First Development (TDD) workflow following constitution principles.

## Execution Steps

### 1. Design
- Define API contract (OpenAPI or Protobuf) → generate Go code
- For OpenAPI: Create/update `api/openapi/<domain>.yaml`
- For Protobuf: Create/update `api/proto/<domain>/v1/<domain>.proto`
- Run code generation (`go generate ./...`)

### 2. Write Tests
- Create table-driven integration tests using generated structs
- Tests MUST use real PostgreSQL (testcontainers) - NO mocking
- Tests MUST go through root mux ServeHTTP (NOT individual handlers)
- Tests MUST use `cmp.Diff()` for struct comparison
- Verify tests FAIL before implementation

```go
func TestFeature(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "table_name")
    
    testCases := []struct {
        name string
        // ... test case fields
    }{
        {name: "US1-AS1: scenario description"},
    }
    
    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            // Test implementation
        })
    }
}
```

### 3. Implement
- Write minimal code to make tests pass
- Services MUST use builder pattern: `NewService(db).Build()`
- ALL validation MUST be in services (handlers are thin wrappers)
- Service methods MUST use protobuf/generated structs (NO primitives)

### 4. Run Tests
// turbo
```bash
go test -v ./...
```

### 5. Run Tests with Race Detector
// turbo
```bash
go test -v -race ./...
```

### 6. Refactor
- Improve code while keeping tests green
- Run tests after each change

### 7. Complete
- Task done only when ALL tests pass
- NO implementation before tests
- Run tests after EVERY code change

## Key Principles

- **Principle INTEGRATION_TESTING**: Integration Testing (No Mocking) - use real PostgreSQL
- **Principle TABLE_DRIVEN**: Table-Driven Design - test cases as slices of structs
- **Principle SERVEHTTP_TESTING**: ServeHTTP Endpoint Testing - test through root mux
- **Principle SCHEMA_FIRST**: Use generated structs with `cmp.Diff()` comparison
- **Principle CONTINUOUS_TESTING**: Continuous Test Verification - run tests after every change
