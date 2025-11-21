# Feature Specification: [FEATURE NAME]

**Feature Branch**: `[###-feature-name]`  
**Created**: [DATE]  
**Status**: Draft  
**Input**: User description: "$ARGUMENTS"

## User Scenarios & Testing *(mandatory)*

<!--
  IMPORTANT: User stories should be PRIORITIZED as user journeys ordered by importance.
  Each user story/journey must be INDEPENDENTLY TESTABLE - meaning if you implement just ONE of them,
  you should still have a viable MVP (Minimum Viable Product) that delivers value.
  
  Assign priorities (P1, P2, P3, etc.) to each story, where P1 is the most critical.
  Think of each story as a standalone slice of functionality that can be:
  - Developed independently
  - Tested independently
  - Deployed independently
  - Demonstrated to users independently
  
  CRITICAL - ACCEPTANCE SCENARIO COVERAGE (Constitution Principle XIII):
  Every acceptance scenario (Given/When/Then) MUST have a corresponding integration test.
  Use naming convention: US#-AS# (e.g., US1-AS1, US1-AS2, US2-AS1)
  This ensures one-to-one mapping between documented requirements and test validation.
-->

### User Story 1 - [Brief Title] (Priority: P1)

[Describe this user journey in plain language]

**Why this priority**: [Explain the value and why it has this priority level]

**Independent Test**: [Describe how this can be tested independently - e.g., "Can be fully tested by [specific action] and delivers [specific value]"]

**Acceptance Scenarios**:

1. **[US1-AS1]** **Given** [initial state], **When** [action], **Then** [expected outcome]
2. **[US1-AS2]** **Given** [initial state], **When** [action], **Then** [expected outcome]

<!--
  NOTE: Each scenario MUST be labeled with [US#-AS#] for traceability to integration tests.
  Integration tests MUST use table-driven design (Principle II) with test case names referencing scenarios.
  Example: TestEnrollmentAcceptanceScenarios with test cases named "US1-AS1: New customer enrolls"
  Tests MUST use protocmp.Transform() for protobuf assertions (Principle VI).
-->

---

### User Story 2 - [Brief Title] (Priority: P2)

[Describe this user journey in plain language]

**Why this priority**: [Explain the value and why it has this priority level]

**Independent Test**: [Describe how this can be tested independently]

**Acceptance Scenarios**:

1. **[US2-AS1]** **Given** [initial state], **When** [action], **Then** [expected outcome]
2. **[US2-AS2]** **Given** [initial state], **When** [action], **Then** [expected outcome]

---

### User Story 3 - [Brief Title] (Priority: P3)

[Describe this user journey in plain language]

**Why this priority**: [Explain the value and why it has this priority level]

**Independent Test**: [Describe how this can be tested independently]

**Acceptance Scenarios**:

1. **[US3-AS1]** **Given** [initial state], **When** [action], **Then** [expected outcome]
2. **[US3-AS2]** **Given** [initial state], **When** [action], **Then** [expected outcome]

---

[Add more user stories as needed, each with an assigned priority]

### Edge Cases

<!--
  ACTION REQUIRED: Edge case coverage is NON-NEGOTIABLE per constitution.
  Each endpoint MUST test the following categories:
-->

**Input Validation**:
- Empty strings, nil values, invalid formats
- SQL injection attempts, XSS payloads
- Oversized inputs, special characters

**Boundary Conditions**:
- Zero values, negative numbers, maximum values
- Empty arrays/slices, nil pointers
- Minimum/maximum field lengths

**Authentication & Authorization**:
- Missing authentication tokens
- Expired tokens, invalid tokens
- Insufficient permissions for operation

**Data State**:
- Non-existent resources (404 scenarios)
- Duplicate entries (conflict scenarios)
- Concurrent modification attempts

**Database Errors**:
- Constraint violations (unique, foreign key, check)
- Transaction conflicts
- Connection failures

**HTTP Specifics**:
- Wrong HTTP methods (GET when POST expected)
- Missing required headers
- Invalid Content-Type
- Malformed JSON/request body

**Observability & Tracing**:
- Verify OpenTracing spans are created for each endpoint
- Verify trace context propagation across service boundaries
- Verify span tags include http.method, http.url, http.status_code
- Verify error spans are tagged with error=true
- Verify child spans are created for service operations (e.g., "ProductService.Create")
- Verify database operations are traced as a single child span per transaction (NOT per SQL query)

**Protobuf Assertions** *(MANDATORY per Principle VI)*:
- ⚠️ **CRITICAL**: ALL protobuf message assertions MUST use `cmp.Diff()` with `protocmp.Transform()`
- ❌ **NEVER** use individual field comparisons (e.g., `if response.Name != expected.Name`)
- ❌ **NEVER** use `==` or `reflect.DeepEqual` for protobuf messages
- ✅ **ALWAYS** build expected from REQUEST data (what you sent), NOT from response data
- ✅ **ONLY** copy generated fields (ID, timestamps) from response to expected
- Verify complete message comparison catches all field differences

**Context Handling**:
- Verify context.Context is passed through all layers (HTTP → Service → Repository)
- Verify context cancellation is handled properly in long-running operations
- Test timeout scenarios with `context.WithTimeout()`
- Test cancellation scenarios with `context.WithCancel()`
- Verify database operations use `db.WithContext(ctx)`

**Error Handling** *(MANDATORY per Principle IX)*:
- Verify errors are wrapped with `fmt.Errorf("%w", err)` (NOT `%v`)
- Verify error messages include contextual information at each layer
- Verify error checking uses `errors.Is()` and `errors.As()` (NOT string comparison)
- Verify HTTP handlers do NOT expose internal error details to clients
- Verify tests validate error chains with `errors.Is()` and `errors.As()`
- ⚠️ **CRITICAL**: ALL defined sentinel errors MUST have test cases
- ⚠️ **CRITICAL**: ALL HTTP error codes MUST have test cases
- ⚠️ **CRITICAL**: Complete error flow MUST be tested (Service → Handler → Client)

## Requirements *(mandatory)*

<!--
  ACTION REQUIRED: The content in this section represents placeholders.
  Fill them out with the right functional requirements.
-->

### Functional Requirements

- **FR-001**: System MUST [specific capability, e.g., "allow users to create accounts"]
- **FR-002**: System MUST [specific capability, e.g., "validate email addresses"]  
- **FR-003**: Users MUST be able to [key interaction, e.g., "reset their password"]
- **FR-004**: System MUST [data requirement, e.g., "persist user preferences"]
- **FR-005**: System MUST [behavior, e.g., "log all security events"]

*Example of marking unclear requirements:*

- **FR-006**: System MUST authenticate users via [NEEDS CLARIFICATION: auth method not specified - email/password, SSO, OAuth?]
- **FR-007**: System MUST retain user data for [NEEDS CLARIFICATION: retention period not specified]

### Key Entities *(include if feature involves data)*

- **[Entity 1]**: [What it represents, key attributes without implementation]
- **[Entity 2]**: [What it represents, relationships to other entities]

## Success Criteria *(mandatory)*

<!--
  ACTION REQUIRED: Define measurable success criteria.
  These must be technology-agnostic and measurable.
-->

### Measurable Outcomes

- **SC-001**: [Measurable metric, e.g., "Users can complete account creation in under 2 minutes"]
- **SC-002**: [Measurable metric, e.g., "System handles 1000 concurrent users without degradation"]
- **SC-003**: [User satisfaction metric, e.g., "90% of users successfully complete primary task on first attempt"]
- **SC-004**: [Business metric, e.g., "Reduce support tickets related to [X] by 50%"]

---

## Error Testing Requirements *(MANDATORY per Constitution Principle IX)*

⚠️ **CRITICAL**: Every defined error MUST have a test case. Untested error paths are production bugs.

### Mandatory Error Testing

All features MUST include comprehensive error testing in a dedicated test file (`tests/integration/error_handling_test.go`):

1. **Service Layer Errors (Sentinel Errors)**
   - Create `TestAllSentinelErrors` test function
   - Test EVERY error defined in `services/errors.go`
   - Verify error wrapping: `fmt.Errorf("context: %w", err)`
   - Verify error checking: `errors.Is(err, expectedError)`
   - Use real database fixtures to trigger errors
   - Table-driven test structure with descriptive names

2. **HTTP Layer Errors (Error Codes)**
   - Create `TestAllHTTPErrorCodes` test function
   - Test EVERY error code defined in `handlers/error_codes.go`
   - Verify HTTP status codes (400, 404, 409, 500, etc.)
   - Verify error response JSON structure
   - Verify `ErrorCode.ServiceErr` field mapping works correctly
   - Use `httptest.ResponseRecorder` for HTTP testing

3. **Complete Error Flow Validation**
   - Create `TestErrorFlowEndToEnd` test function
   - Verify: Service (sentinel error) → Handler (HTTP code) → Client (JSON response)
   - Test context errors (cancellation → 499, timeout → 504)
   - Validate error message propagation

### Error Test Structure Template

```go
// File: tests/integration/error_handling_test.go

func TestAllSentinelErrors(t *testing.T) {
    // Setup test database
    db, cleanup := testutil.SetupTestDB(t)
    defer cleanup()
    defer testutil.TruncateTables(db, "all", "tables")
    
    testCases := []struct {
        name          string
        serviceCall   func(context.Context) error
        expectedError error
        description   string
    }{
        {
            name: "ErrProductNotFound",
            serviceCall: func(ctx context.Context) error {
                _, err := productService.Get(ctx, &pb.GetProductRequest{
                    Id: "550e8400-e29b-41d4-a716-446655440000",
                })
                return err
            },
            expectedError: services.ErrProductNotFound,
            description:   "Product not found error when getting non-existent product",
        },
        // ONE test case per error in services/errors.go (MANDATORY)
    }
    
    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            err := tc.serviceCall(context.Background())
            
            if err == nil {
                t.Fatalf("Expected error %v, but got nil", tc.expectedError)
            }
            
            if !errors.Is(err, tc.expectedError) {
                t.Errorf("Expected error chain to contain %v, got: %v", tc.expectedError, err)
            }
        })
    }
}

func TestAllHTTPErrorCodes(t *testing.T) {
    testCases := []struct {
        name             string
        httpCall         func() *httptest.ResponseRecorder
        expectedStatus   int
        expectedErrorCode string
        description      string
    }{
        {
            name: "ProductNotFound",
            httpCall: func() *httptest.ResponseRecorder {
                req := httptest.NewRequest(http.MethodGet, "/api/v1/products/550e8400-e29b-41d4-a716-446655440000", nil)
                rec := httptest.NewRecorder()
                productHandler.Get(rec, req)
                return rec
            },
            expectedStatus:   http.StatusNotFound,
            expectedErrorCode: "PRODUCT_NOT_FOUND",
            description:      "Product not found returns 404 with correct error code",
        },
        // ONE test case per error code in handlers/error_codes.go (MANDATORY)
    }
    
    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            rec := tc.httpCall()
            
            if rec.Code != tc.expectedStatus {
                t.Errorf("Expected status %d, got %d", tc.expectedStatus, rec.Code)
            }
            
            var errorResponse pb.ErrorResponse
            json.NewDecoder(rec.Body).Decode(&errorResponse)
            
            if errorResponse.Code != tc.expectedErrorCode {
                t.Errorf("Expected error code '%s', got '%s'", tc.expectedErrorCode, errorResponse.Code)
            }
        })
    }
}
```

### Checklist for Error Testing

Before feature is complete:
- [ ] Every error in `services/errors.go` has a test case
- [ ] Every error code in `handlers/error_codes.go` has a test case
- [ ] Complete error flow validated (Service → Handler → Client)
- [ ] Context errors tested (cancellation, timeout)
- [ ] All error tests pass: `go test -v -run "TestAll.*Errors"`
- [ ] Zero untested error paths remain
