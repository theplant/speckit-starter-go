---
description: Analyze test coverage and recover untested code paths following constitution principles.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Analyze test coverage and recover untested code paths following constitution principles.

## Execution Steps

### Step 1: Identify Gaps

Run coverage analysis with `-coverpkg` to count cross-package calls:
// turbo
```bash
go test -coverprofile=coverage.out -coverpkg=./... ./...
```

**IMPORTANT**: The `-coverpkg=./...` flag ensures that when tests in one package call code in another package, that code is counted as covered. Without this flag, only code in the same package as the test file is counted.

View coverage summary:
// turbo
```bash
go tool cover -func=coverage.out
```

View detailed HTML report:
```bash
go tool cover -html=coverage.out -o coverage.html
```

### Step 2: Analyze Gaps

1. Read files with low coverage to understand untested branches
2. Identify:
   - Error paths not tested
   - Edge cases missed
   - Conditionals with untested branches
   - Validation paths not exercised

### Step 3: Write Tests

Add table-driven test cases covering identified gaps:

```go
func TestFeature_EdgeCases(t *testing.T) {
    testCases := []struct {
        name     string
        input    *pb.Request
        wantErr  error
        wantCode int
    }{
        {
            name:     "empty required field returns validation error",
            input:    &pb.Request{Name: ""},
            wantErr:  services.ErrMissingRequired,
            wantCode: http.StatusBadRequest,
        },
        {
            name:     "duplicate key returns conflict error",
            input:    &pb.Request{Name: "existing"},
            wantErr:  services.ErrDuplicateSKU,
            wantCode: http.StatusConflict,
        },
        // Add more edge cases...
    }
    
    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            // Test implementation following Principles I-VIII
        })
    }
}
```

### Step 4: Verify Coverage Improvement

Re-run coverage to confirm gaps are closed:
// turbo
```bash
go test -coverprofile=coverage.out -coverpkg=./... ./...
go tool cover -func=coverage.out
```

**Target**: >80% coverage for business logic

### Step 5: Clean Dead Code

If code cannot be reached legitimately:
- Remove it rather than exempting
- Dead code indicates design issues

### Coverage Checklist

Every endpoint MUST test (Principle EDGE_CASE_COVERAGE):

- [ ] **Input validation**: Empty/nil values, invalid formats, SQL injection, XSS
- [ ] **Boundary conditions**: Zero/negative/max values, empty arrays, nil pointers
- [ ] **Auth**: Missing/expired/invalid tokens, insufficient permissions
- [ ] **Data state**: 404s, conflicts, concurrent modifications
- [ ] **Database**: Constraint violations, foreign key failures, transaction conflicts
- [ ] **HTTP**: Wrong methods, missing headers, invalid content-types, malformed JSON

### AI Agent Requirements

- Run `go test -coverprofile=coverage.out -coverpkg=./... ./...` to identify gaps
- Read files with low coverage to understand untested branches
- Write table-driven test cases following Principles INTEGRATION_TESTING through ACCEPTANCE_COVERAGE
- Re-run coverage to verify gaps are closed (target >80%)
- Remove unreachable code rather than exempting it
