# Testing Guidelines Appendix

**Part of**: [Go Project Constitution](../constitution.md)  
**Covers**: Principles I-X

This appendix provides detailed implementation guidelines for all testing principles defined in the constitution.

---

## Testing Principles

### I. Integration Testing First (No Mocking)

All tests MUST be integration tests that interact with real dependencies:
- Tests MUST use real PostgreSQL database connections
- NO mocking of database calls, HTTP clients, or external services
- Tests MUST prepare fixture data directly in the database
- Test database MUST be isolated per test run
- Each test MUST use database truncation for isolation (truncate tables after test)

**Rationale**: Integration tests catch real-world issues that unit tests with mocks cannot, including database constraint violations, connection pooling issues, transaction handling bugs, and serialization problems.

### II. Table-Driven Test Design

All tests MUST follow table-driven test patterns:
- Tests MUST define test cases as slices of structs with inputs and expected outputs
- Each test case MUST have a descriptive name field
- Test runner MUST iterate over cases using `t.Run(testCase.name, func(t *testing.T) {...})`
- Shared setup/teardown logic MUST be extracted to helper functions
- Test tables MUST be readable and maintainable

**Rationale**: Table-driven tests reduce code duplication, make test cases easy to add/modify, improve test readability, and enable comprehensive scenario coverage with minimal code.

### III. Edge Case Coverage (NON-NEGOTIABLE)

Every API endpoint MUST test comprehensive edge cases:
- **Input validation**: Empty strings, nil values, invalid formats, SQL injection attempts, XSS payloads
- **Boundary conditions**: Zero values, negative numbers, maximum values, empty arrays, nil pointers
- **Authentication/Authorization**: Missing tokens, expired tokens, invalid tokens, insufficient permissions
- **Data state**: Non-existent resources (404), duplicate entries (conflict), concurrent modifications
- **Database errors**: Constraint violations, foreign key failures, transaction conflicts
- **HTTP specifics**: Wrong methods, missing headers, invalid content-types, malformed JSON
- Edge cases MUST be documented in test case names

**Rationale**: Production systems face unexpected inputs and conditions. Comprehensive edge case testing prevents security vulnerabilities, data corruption, and runtime panics.

### IV. Real Database Fixtures

Test data MUST be prepared using real database operations:
- Fixture data MUST be inserted using GORM to real test database
- NO in-memory mocks or fake repositories
- Fixtures MUST represent realistic production data scenarios
- Complex fixtures (with foreign keys, relationships) MUST be created with helper functions
- Fixture helpers MUST return created entities for test verification
- Fixtures MUST handle database constraints correctly

**Rationale**: Real database fixtures ensure tests validate actual database behavior including constraints, triggers, indexes, and query performance.

### V. ServeHTTP Endpoint Testing

API endpoints MUST be tested via ServeHTTP interface:
- Tests MUST create `httptest.ResponseRecorder` to capture responses
- Tests MUST construct `*http.Request` with proper method, URL, headers, and body
- Tests MUST pass requests through actual HTTP handler chains (middleware included)
- Tests MUST verify response status codes, headers, and body content
- JSON responses MUST be parsed and validated structurally
- Tests MUST NOT bypass HTTP layer by calling service functions directly
- **Tests MUST call the root routes handler ServeHTTP method** (NOT individual handler methods)
- **URL routing and path handling MUST be tested** by calling the root HTTP mux
- **Production API and tests MUST use identical routing configuration** from shared routes package
- **HTTP path patterns MUST be used** (e.g., `"POST /api/v1/products/{productID}"`) for type-safe routing
- **Path parameters MUST be extracted using `r.PathValue()`** (NOT string manipulation of `r.URL.Path`)

**Rationale**: Testing through ServeHTTP ensures complete HTTP stack validation including routing, middleware, request parsing, content negotiation, error handling, and response formatting. Using identical routing configuration between production and tests prevents routing inconsistencies and ensures test accuracy. HTTP path patterns provide type-safe parameter extraction and eliminate routing bugs that bypass traditional testing. The `PathValue()` method ensures proper parameter extraction that matches the path pattern definitions, preventing parsing errors and maintaining consistency between routing and parameter handling.

**CRITICAL - Why Call Root Mux ServeHTTP, Not Individual Handlers**:

The Constitution requires tests to call `mux.ServeHTTP(rec, req)` instead of `handler.Create(rec, req)` because:

1. **Routing is tested**: Calling individual handlers bypasses URL routing entirely. The test never validates:
   - Route patterns match correctly
   - HTTP method matching works (POST vs GET)
   - Path parameters are extracted properly
   - Route conflicts are caught
   - URL paths route to correct handlers

2. **Production parity**: If production uses `mux.ServeHTTP`, but tests call handlers directly, you're testing different code paths. Tests must exercise the same execution path as production.

3. **Middleware is included**: Root mux ServeHTTP executes middleware chains (logging, auth, CORS, etc.). Calling handlers directly skips all middleware, leaving it untested.

4. **Real-world HTTP flow**: Browsers and API clients call the root endpoint, not individual handler methods. Tests should simulate actual usage.

**Example - What Gets Missed**:
```go
// ❌ WRONG: Calling handler method directly
handler.Create(rec, req)
// Bypasses: routing, method matching, path patterns, middleware
// Test passes even if route registration is broken!

// ✅ CORRECT: Calling root mux
mux.ServeHTTP(rec, req)
// Tests: routing, method matching, path patterns, middleware, handler
// Test fails if route registration is wrong (catches real bugs!)
```

**Real Bug Example** - Why This Matters:
```go
// Production code registers route
mux.HandleFunc("POST /api/products", handler.Create)

// Test calls handler directly (WRONG)
handler.Create(rec, req)  // ✅ Test passes

// Deploy to production
// Route was accidentally registered as:
mux.HandleFunc("POST /api/product", handler.Create)  // Typo: "product" not "products"

// Production: 404 Not Found (route doesn't match!)
// Tests: All green (because they bypassed routing)
```

**Proper Testing Through Root Mux**:
```go
// Test calls root mux (CORRECT)
req := httptest.NewRequest("POST", "/api/products", body)
mux.ServeHTTP(rec, req)  // ✅ Test fails if route is wrong!
```

This principle catches routing bugs before production, not after.

**Testing Path Parameters**:
```go
// Test GET endpoint with path parameter
func TestProductAPI_Get_Success(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    // Create fixture
    existingProduct := createTestProduct(db, map[string]interface{}{
        "name": "Test Product",
        "sku":  "TEST-001",
    })
    
    // Setup routes
    service := NewProductService(db).Build()
    mux := SetupRoutes(service)
    
    // ✅ CORRECT: Test through root mux with path parameter in URL
    req := httptest.NewRequest("GET", "/api/products/"+existingProduct.ID, nil)
    rec := httptest.NewRecorder()
    
    mux.ServeHTTP(rec, req)  // Tests routing extracts ID correctly
    
    if rec.Code != http.StatusOK {
        t.Fatalf("Expected status %d, got %d", http.StatusOK, rec.Code)
    }
    
    var response pb.Product
    json.NewDecoder(rec.Body).Decode(&response)
    
    // Verify correct product was retrieved via path parameter
    expected := &pb.Product{
        Id:   existingProduct.ID,   // From database fixture
        Name: "Test Product",       // From database fixture
        Sku:  "TEST-001",           // From database fixture
    }
    
    if diff := cmp.Diff(expected, &response, protocmp.Transform()); diff != "" {
        t.Errorf("Response mismatch (-want +got):\n%s", diff)
    }
}

// Test wrong path parameter (should 404)
func TestProductAPI_Get_NotFound(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    
    service := NewProductService(db)
    mux := SetupRoutes(service)
    
    // Request non-existent product ID
    req := httptest.NewRequest("GET", "/api/products/00000000-0000-0000-0000-000000000000", nil)
    rec := httptest.NewRecorder()
    
    mux.ServeHTTP(rec, req)
    
    // Should return 404 (routing works, handler executed, resource not found)
    if rec.Code != http.StatusNotFound {
        t.Errorf("Expected status %d, got %d", http.StatusNotFound, rec.Code)
    }
}
```

### VI. Protobuf Data Structures

All public API data structures MUST be defined in Protocol Buffers:
- API request and response types MUST be defined in `.proto` files
- Tests MUST use protobuf-generated structs, NOT `map[string]interface{}`
- NO use of untyped maps for request/response handling in tests or production code
- Protobuf definitions MUST be the single source of truth for API contracts
- Generated Go structs MUST be used for JSON marshaling/unmarshaling
- Protobuf messages MUST include field validation rules (e.g., `validate.rules`)
- All API changes MUST update the corresponding `.proto` files first
- Tests MUST use proper protobuf comparison packages for assertions (e.g., `protocmp` with `google/go-cmp`)
- Tests MUST NOT use standard `==` or `reflect.DeepEqual` for protobuf message comparison
- Tests MUST NOT use individual field comparisons (e.g., `if response.Name != expected.Name`) for protobuf messages
- ALL protobuf message assertions in tests MUST use `cmp.Diff()` with `protocmp.Transform()` to compare entire messages
- **Expected test data MUST be derived from TEST FIXTURES (input data), NOT from RESPONSE data**
- **Test fixtures include: request payload data, database fixture data, configuration values, test constants**
- **ALWAYS try your best to derive expected values from test fixtures before considering response values**
- **Copying response data into expected values defeats the purpose of testing (test will always pass)**
- **Use response values ONLY for truly random/generated fields that cannot be derived from fixtures**
- Generated fields (ID, timestamps, secure tokens) are the ONLY exception - these can be copied from response
- Individual field checks are ONLY acceptable for non-protobuf types (e.g., checking if a string ID is not empty before comparison)

**Rationale**: Protobuf provides compile-time type safety, eliminates runtime type assertion errors, enables automatic validation, supports multiple language clients, enforces schema-first API design, and prevents the fragile `map[string]interface{}` pattern that loses type information and requires extensive runtime validation. Proper protobuf comparison ensures correct field comparison including unknown fields, extensions, and proto semantics. Individual field comparisons miss structural differences, ignore unknown fields, and fail to validate the complete message structure, leading to incomplete test coverage.

**Examples**:
```protobuf
// api/product.proto
message CreateProductRequest {
  string name = 1 [(validate.rules).string.min_len = 1];
  string sku = 2 [(validate.rules).string.pattern = "^[a-zA-Z0-9_-]+$"];
  string description = 3;
  map<string, AttributeValue> attributes = 4;
}

message Product {
  string id = 1;
  string name = 2;
  string sku = 3;
  string description = 4;
  map<string, AttributeValue> attributes = 5;
  google.protobuf.Timestamp created_at = 6;
  google.protobuf.Timestamp updated_at = 7;
}
```

**Test Usage**:
```go
// CORRECT: Use protobuf structs
req := &pb.CreateProductRequest{
    Name: "Test Product",
    Sku:  "TEST-001",
}

// WRONG: Do not use maps
req := map[string]interface{}{
    "name": "Test Product",
    "sku":  "TEST-001",
}
```

**Test Assertions**:
```go
import (
    "testing"
    "github.com/google/go-cmp/cmp"
    "google.golang.org/protobuf/testing/protocmp"
)

// CORRECT: Build expected from TEST FIXTURES (request data + database fixtures)
var response pb.CreateProductResponse
json.NewDecoder(rec.Body).Decode(&response)

// Derive expected from FIXTURES - request data (what you sent) + DB fixtures (what you created)
expectedResponse := &pb.CreateProductResponse{
    Product: &pb.Product{
        Id:          response.Product.Id,        // Use generated ID from response (random)
        Name:        requestData.Name,           // From REQUEST fixture (what you sent)
        Sku:         requestData.Sku,            // From REQUEST fixture
        Description: requestData.Description,    // From REQUEST fixture
        Price:       requestData.Price,          // From REQUEST fixture
        CreatedAt:   response.Product.CreatedAt, // Use generated timestamp (random)
        UpdatedAt:   response.Product.UpdatedAt, // Use generated timestamp (random)
    },
}

// Compare entire messages - validates fixtures match response
if diff := cmp.Diff(expectedResponse, &response, protocmp.Transform()); diff != "" {
    t.Errorf("Response mismatch (-want +got):\n%s", diff)
}

// Example with DATABASE FIXTURE data (for endpoints that reference existing data)
// CORRECT: Derive from database fixture you created in test setup
category := testutil.CreateTestCategory(db, map[string]interface{}{
    "name": "Electronics",
    "slug": "electronics",
})

var getResponse pb.GetProductResponse
json.NewDecoder(rec.Body).Decode(&getResponse)

expectedGetResponse := &pb.GetProductResponse{
    Product: &pb.Product{
        Id:           getResponse.Product.Id,       // Generated (random)
        Name:         "Laptop",                     // From REQUEST fixture
        CategoryId:   category.ID,                  // From DATABASE fixture (created above)
        CategoryName: "Electronics",                // From DATABASE fixture
        CategorySlug: "electronics",                // From DATABASE fixture
        CreatedAt:    getResponse.Product.CreatedAt, // Generated (random)
    },
}

if diff := cmp.Diff(expectedGetResponse, &getResponse, protocmp.Transform()); diff != "" {
    t.Errorf("Response mismatch (-want +got):\n%s", diff)
}

// ALTERNATIVE: Ignore generated fields in comparison
if diff := cmp.Diff(expectedResponse, &response, 
    protocmp.Transform(),
    protocmp.IgnoreFields(&pb.Product{}, "id", "created_at", "updated_at"),
); diff != "" {
    t.Errorf("Response mismatch (-want +got):\n%s", diff)
}

// WRONG: Using response data as expected data
expectedResponse := &pb.CreateProductResponse{
    Product: &pb.Product{
        Name:  response.Product.Name,  // WRONG! Copying from response
        Price: response.Product.Price, // WRONG! This doesn't test anything!
    },
}
// This test will always pass even if API returns wrong data!

// WRONG: Do not use individual field comparisons
if response.Product.Name != expectedResponse.Product.Name { // Misses other fields!
    t.Error("name mismatch")
}

// WRONG: Do not use == or reflect.DeepEqual
if actual != expected { // Incorrect for protobuf
    t.Error("mismatch")
}
```

**Handling Generated Fields**:
```go
// Option 1: Use generated values from response (for ID, timestamps)
expected := &pb.Product{
    Id:        response.Product.Id,        // Can't predict generated UUID
    Name:      "Expected Name",            // From request
    CreatedAt: response.Product.CreatedAt, // Can't predict exact timestamp
}

// Option 2: Ignore generated fields in comparison
expected := &pb.Product{
    Name:  "Expected Name",
    Price: 99.99,
}
if diff := cmp.Diff(expected, actual, 
    protocmp.Transform(),
    protocmp.IgnoreFields(&pb.Product{}, "id", "created_at", "updated_at"),
); diff != "" {
    t.Errorf("Mismatch: %s", diff)
}

// Option 3: Validate generated fields separately, then nil them out
if response.Id == "" {
    t.Error("Expected ID to be generated")
}
if response.CreatedAt == nil {
    t.Error("Expected CreatedAt to be set")
}
// Then set to nil for comparison
expected.Id = response.Id
expected.CreatedAt = response.CreatedAt
```

#### What Qualifies as "Truly Random/Generated"

ONLY four field types may copy response values in test assertions:

1. **Database UUIDs**: `gen_random_uuid()` generated fields (`id`, `customer_id`)
2. **System timestamps**: Operation execution time (`created_at`, `updated_at`, `enrolled_at`)
3. **Crypto-random values**: `crypto/rand` generated tokens (`membership_number`, `referral_code`)
4. **Unpredictable calculations**: Checksums, hashes with random inputs

**Everything else MUST use fixture values**, including:
- Database/fixture defaults
- Enum values and constants
- Configuration values
- Request payload data
- Empty/null/zero values

#### The Anti-Pattern: Lazy Response Copying

**DON'T do this:**

```go
expected := &pb.Tier{
    Name:        "Base",
    Description: resp.Tier.Description,    // ❌ Copying fixture value - test always passes!
}
// Comment: "can't derive" (FALSE - it's in CreateTestTier default)
```

**Why this breaks testing:**
- Test passes even if API returns garbage
- No actual validation happens
- Defeats the purpose of testing

**DO this instead:**

```go
// Step 1: Read testutil/fixtures.go to find CreateTestTier() default
// Step 2: Use that default value
expected := &pb.Tier{
    Name:        "Base",
    Description: "Base tier for testing",  // From fixtures.go:21
}
```

#### Code Review Checklist

For each test, reviewers MUST verify:

**✅ Fixture Investigation:**
- [ ] Read `testutil/fixtures.go` to find defaults
- [ ] Check request payload for input values
- [ ] Document fixture sources in comments

**✅ Expected Values:**
- [ ] UUIDs/timestamps/crypto-rand use response ✓
- [ ] All other fields use fixture values ✓
- [ ] No `resp.Field` without justification

**🚩 Red Flags (reject PR):**
- Comments claiming "can't derive" for non-random fields
- Multiple fields copying from response
- Missing fixture source documentation
- Test written without reading fixture code

#### AI Agent Requirements

Before writing test assertions, AI agents MUST:

1. **Read fixtures first**: `read_file("testutil/fixtures.go")` to find `CreateTestXxx()` defaults
2. **Document sources**: Add comments showing where each value comes from
3. **Use fixture values**: Only use `resp.Field` for the 4 random types above
4. **Never claim "can't derive"** without proof of true randomness

**Prohibited shortcuts:**
- ❌ Copying response without reading fixtures
- ❌ Claiming "generated" without evidence
- ❌ Lazy response copying to make tests pass

#### Example: Correct Fixture Derivation

```go
// Step 1: Read testutil/fixtures.go CreateTestTier()
// Found: Description: "Base tier for testing" at line 21

// Step 2: Build expected from fixtures
expected := &pb.Customer{
    // Random fields (use response):
    Id:               resp.Customer.Id,               // Database UUID
    MembershipNumber: resp.Customer.MembershipNumber, // crypto/rand
    ReferralCode:     resp.Customer.ReferralCode,     // crypto/rand
    EnrolledAt:       resp.Customer.EnrolledAt,       // Timestamp
    CreatedAt:        resp.Customer.CreatedAt,        // Timestamp
    UpdatedAt:        resp.Customer.UpdatedAt,        // Timestamp
    
    // Fixture fields (derive from sources):
    AccountId:       "user-001",                // From REQUEST
    CurrentBalance:  0,                         // Known rule
    Tier: &pb.Tier{
        Id:          baseTier.ID,               // DATABASE fixture
        Name:        "Base",                    // fixtures.go:16
        Level:       0,                         // fixtures.go:17
        Description: "Base tier for testing",   // fixtures.go:21 ✓
    },
}
```

#### Enforcement: Reject PRs With Violations

**Reviewers MUST reject PRs containing:**
- Non-random fields using `resp.Field` without justification
- Comments claiming "can't derive" for fixture/config data
- Missing fixture source documentation
- Tests written without reading fixture code

**Response for violations:**

```markdown
❌ Constitution Principle VI Violation

Field: `Description: resp.Customer.Tier.Description`
Claim: "can't derive"

Fix required:
1. Read testutil/fixtures.go CreateTestTier()
2. Find Description default value
3. Use that value: `Description: "Base tier for testing", // fixtures.go:21`
```

**Escalation:** Repeated violations require AI agent process review.

### VII. Continuous Test Verification

All code changes MUST be verified by running tests immediately after implementation:
- Tests MUST be executed after every code change before committing
- Tests MUST pass before any commit to version control
- Integration tests MUST be run locally using testcontainers before pushing
- Developers MUST verify tests pass in their local environment first
- CI/CD pipeline MUST block merges if tests fail
- Tests MUST be run after bug fixes to verify the fix and prevent regressions
- Tests MUST be run after refactoring to ensure behavior is preserved
- NO code changes MUST be committed without running the test suite
- Test execution MUST be automated in pre-commit hooks where possible
- Test failures MUST be fixed immediately before proceeding with new work
- Developers MUST NOT skip or disable tests to "temporarily" fix CI/CD
- Test execution time MUST be monitored and optimized if it impacts development velocity

**Rationale**: Running tests continuously after code changes is the only reliable way to catch regressions early, ensure code quality gates are maintained, and prevent broken code from reaching production. This practice creates a tight feedback loop that catches issues when context is fresh and fixes are cheap. Without continuous test verification, even comprehensive test suites become worthless because passing tests from yesterday don't guarantee today's changes work correctly. This principle transforms testing from a checkbox activity into a living quality gate that protects the codebase at every step. Test-first development (TDD) is valuable, but continuous verification ensures the tests continue passing as the code evolves.

**AI Implementation Requirement**:
- AI agents MUST run the full test suite (`go test -v ./...`) after making any code changes
- AI agents MUST verify all tests pass before completing the task
- AI agents MUST report test failures and fix them before proceeding
- AI agents MUST run tests with race detector (`go test -v -race ./...`) for concurrency safety
- AI agents MUST NOT skip or disable failing tests to complete a task
- AI agents MUST treat test failures as blocking issues that require immediate resolution

**Test Execution Requirements**:
- Full test suite MUST pass before task completion
- Tests MUST include integration tests with real database (testcontainers)
- Race detector MUST be used to catch concurrency issues
- Test coverage MUST be maintained or improved
- Flaky tests MUST be fixed immediately (not ignored or re-run)

### VIII. Root Cause Tracing (Debugging Discipline)

When problems occur during development, root cause analysis MUST be performed before implementing fixes:
- Problems MUST be traced backward through the call chain to find the original trigger
- Symptoms MUST be distinguished from root causes
- Fixes MUST address the source of the problem, NOT work around symptoms
- Test cases MUST NOT be removed or weakened to make tests pass
- Debuggers and logging MUST be used to understand control flow
- Multiple potential causes MUST be systematically eliminated
- Documentation MUST be updated to prevent similar issues
- Root cause MUST be verified through testing before closing the issue

**Rationale**: Superficial fixes create technical debt and hide underlying architectural problems. Root cause analysis ensures problems are solved at their source, preventing recurrence and maintaining system integrity. This discipline transforms debugging from firefighting into systematic problem-solving that improves overall code quality.

**Debugging Process**:
1. **Reproduce**: Create reliable reproduction case
2. **Observe**: Gather evidence through logs, debugger, tests
3. **Hypothesize**: Form theories about root cause
4. **Test**: Design experiments to validate/invalidate hypotheses
5. **Fix**: Implement fix addressing root cause
6. **Verify**: Ensure fix works and doesn't break existing functionality
7. **Document**: Update docs/tests to prevent regression

**AI Implementation Requirement**:
- AI agents MUST perform root cause analysis before implementing fixes
- AI agents MUST NOT implement superficial workarounds
- AI agents MUST document the root cause analysis process
- AI agents MUST update tests to prevent regression of root causes

### IX. Acceptance Scenario Coverage (Spec-to-Test Mapping)

Every user scenario in specifications MUST have corresponding automated tests:
- Each acceptance scenario (US#-AS#) in spec.md MUST have a test case
- Test case names MUST reference source scenarios (e.g., "US1-AS1: New customer enrolls")
- Test functions MUST use table-driven design with test case structs (Principle II)
- Each test case struct MUST include `name` field with scenario ID
- Tests MUST validate complete "Then" clause, not partial behavior
- No untested scenarios MUST exist (or be explicitly deferred with justification)
- Test coverage analysis MUST verify all scenarios are tested
- AI agents MUST flag any scenarios that cannot be tested with justification
- AI agents MUST update tests when specifications change to add/modify scenarios

**Rationale**: Specifications define expected behavior, but only automated tests guarantee that behavior is maintained. Acceptance scenario coverage bridges the gap between business requirements and implementation, ensuring the system actually does what the specifications claim. Without this mapping, specifications become documentation that drifts from reality, leading to false assumptions and undetected regressions.

**Test Case Structure**:
```go
func TestUserAcceptanceScenarios(t *testing.T) {
    testCases := []struct {
        name     string
        scenario string
        given    func() // Setup
        when     func() // Action
        then     func() // Assertion
    }{
        {
            name:     "US1-AS1: New customer enrolls",
            scenario: "Given a new user, When they complete enrollment, Then account is created",
            given: func() { /* setup */ },
            when:  func() { /* action */ },
            then:  func() { /* verify */ },
        },
    }
    // ... test execution
}
```

**Code Review Checklist**:
- [ ] Every acceptance scenario in spec.md has a corresponding test case
- [ ] Test case names in table reference source scenarios (e.g., "US1-AS1: New customer enrolls")
- [ ] Test function uses table-driven design with test case structs
- [ ] Test case struct includes `name` field with scenario ID (US#-AS#)
- [ ] No untested scenarios exist (or are explicitly deferred with justification)
- [ ] Test validates complete "Then" clause, not partial behavior
- [ ] Traceability matrix is up to date (optional but recommended)

### X. Test Coverage & Gap Analysis

Test coverage analysis MUST be used to identify and close testing gaps:
- Developers MUST run `go test -cover` (or `go tool cover` for visualization) to identify untested code paths
- Uncovered code paths identified by coverage reports MUST be covered by new tests
- Coverage gaps MUST be analyzed to determine if they represent dead code or missing tests
- Missing tests MUST be added to reach acceptable coverage levels (typically >80% for business logic)
- Code that cannot be reached (dead code) SHOULD be removed rather than exempted

**Rationale**: Coverage analysis provides objective data on testing completeness. It reveals edge cases, error paths, and logic branches that were missed during TDD. While 100% coverage is not always practical, coverage analysis ensures that critical business logic and error handling paths are not left untested.


## Development Workflow

### Test-First Development (TDD)

1. **Design Phase**: Define API contract in `.proto` files (request/response messages)
2. **Generate Code**: Run `protoc` to generate Go structs from protobuf definitions
3. **Write Tests**: Create table-driven integration tests using protobuf structs (NOT maps)
4. **Verify Failure**: Run tests to confirm they fail (red phase)
5. **Review Tests**: Review test design with team/lead before implementation
6. **Implement**: Write minimal code to make tests pass (green phase)
7. **Run Tests**: Execute full test suite immediately after implementation (MANDATORY per Principle VII)
8. **Verify Success**: Confirm all tests pass (green phase)
9. **Refactor**: Improve code quality while keeping tests green
10. **Run Tests Again**: Execute tests after each refactoring change (MANDATORY per Principle VII)
11. **Complete Task**: Task is only complete when all tests pass
12. **No Implementation Before Tests**: Code written before test approval MUST be discarded

**Critical**: Steps 7, 8, 10, and 11 enforce Principle VII (Continuous Test Verification). AI agents MUST run tests after EVERY code change and verify they pass before task completion.

### Protobuf Workflow

1. **Define Schema**: Create or update `.proto` files in `api/` directory
2. **Generate Code**: Run `go generate` to create Go structs (add `//go:generate` directives)
3. **Use in Code**: Import generated packages, use typed structs throughout
4. **Validate**: Use protoc-gen-validate for automatic field validation
5. **Version**: Use protobuf field numbers consistently (never reuse deleted field numbers)

### Test Database Management

- Each developer MUST have Docker Engine (Linux) or Docker Desktop (Mac/Windows) installed
- Test suite MUST use testcontainers-go library for PostgreSQL container management
- Test suite MUST use `testcontainers.PostgresContainer` for automatic lifecycle management
- Container startup, port allocation, and cleanup are handled automatically by testcontainers
- Test database schema MUST match production schema via GORM AutoMigrate
- Container cleanup MUST use `defer container.Terminate(ctx)` pattern
- CI/CD environments MUST have Docker daemon available for testcontainers

### Test Isolation Strategy

**Database Truncation** - The single, simple approach for all tests.

**How it works:**
- Tests run with real database and commit transactions normally
- After each test completes, truncate all modified tables to clean up
- Use `defer` pattern to ensure cleanup happens even on test failure

**Requirements:**
- Tests MUST truncate all relevant tables after each test using `defer`
- Truncation MUST use `CASCADE` to handle foreign key constraints
- Truncate in reverse dependency order (children before parents) for safety
- Use helper function to centralize truncation logic

**Benefits:**
- ✅ Works with any code structure (no special patterns needed)
- ✅ Tests actual production behavior (with real commits)
- ✅ Simple and reliable
- ✅ No limitations or gotchas
- ✅ Write production code naturally
- ✅ Fast enough (~1-5ms overhead per test)

### Architecture Overview for Testing

**Data Flow**: HTTP Request → Handler → Service → GORM Model → Database → GORM Model → Protobuf → HTTP Response

**Key Principles**:
1. **HTTP Layer**: Receives/sends JSON (serialized protobuf structs)
2. **Handler Layer**: Thin wrapper, delegates to service
3. **Service Layer**: Business logic, accepts/returns protobuf types
4. **Model Layer**: Internal GORM models for database operations (NEVER exposed in APIs)
5. **Database Layer**: PostgreSQL with GORM

**Why This Matters for Tests**:
- Tests exercise full stack via **root mux ServeHTTP** (NOT individual handler methods)
- Tests use protobuf for assertions (NOT internal models)
- Tests verify database state using internal models (for verification only)
- Services are testable independently (accept/return protobuf)
- **Production and tests share identical routing configuration** (SetupRoutes function)

**Testing Flow**:
```
Test Request → mux.ServeHTTP() → [Routing Layer] → [Middleware] → Handler → Service → Database
                                      ↑
                        This is what gets tested!
```

**If you call handler methods directly, you skip routing and middleware** ❌

**Example Testcontainers Setup**:
```go
import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "net/http/httptest"
    "testing"
    
    "github.com/google/go-cmp/cmp"
    "google.golang.org/protobuf/testing/protocmp"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
    
    pb "yourapp/api/gen/pim/v1"  // Protobuf generated types
)

// Setup PostgreSQL test container using testcontainers
func setupTestDB(t *testing.T) (*gorm.DB, func()) {
    ctx := context.Background()
    
    // Create PostgreSQL container
    pgContainer, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:15-alpine"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("postgres"),
        postgres.WithPassword("postgres"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2),
        ),
    )
    if err != nil {
        t.Fatalf("Failed to start PostgreSQL container: %v", err)
    }
    
    // Cleanup function
    cleanup := func() {
        if err := pgContainer.Terminate(ctx); err != nil {
            t.Logf("Failed to terminate container: %v", err)
        }
    }
    
    // Get connection string
    connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
        cleanup()
        t.Fatalf("Failed to get connection string: %v", err)
    }
    
    // Connect using GORM
    db, err := gorm.Open(postgres.Open(connStr), &gorm.Config{})
    if err != nil {
        cleanup()
        t.Fatalf("Failed to connect to database: %v", err)
    }
    
    // Run GORM AutoMigrate with internal models
    // Note: Use your actual internal models here
    if err := db.AutoMigrate(&Product{} /* add other models */); err != nil {
        cleanup()
        t.Fatalf("Failed to run migrations: %v", err)
    }
    
    return db, cleanup
}

// Create helper function for truncation (reusable across all tests)
func truncateTables(db *gorm.DB, tables ...string) {
    // Truncate in reverse order (children before parents)
    for i := len(tables) - 1; i >= 0; i-- {
        db.Exec(fmt.Sprintf("TRUNCATE TABLE %s CASCADE", tables[i]))
    }
}

// Example test - CORRECT ServeHTTP usage through root mux
func TestProductCreate(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    
    // Truncate tables after test (in reverse dependency order)
    defer func() {
        db.Exec("TRUNCATE TABLE order_items CASCADE")
        db.Exec("TRUNCATE TABLE orders CASCADE")
        db.Exec("TRUNCATE TABLE products CASCADE")
    }()
    
    // Setup service and routes (shared configuration)
    service := NewProductService(db).Build()
    mux := SetupRoutes(service)  // ✅ Use shared routes setup
    
    // Create HTTP request with protobuf data
    reqData := &pb.CreateProductRequest{
        Name: "Test Product",
        Sku:  "TEST-001",
    }
    body, _ := json.Marshal(reqData)
    req := httptest.NewRequest("POST", "/api/products", bytes.NewReader(body))
    rec := httptest.NewRecorder()
    
    // ✅ CORRECT: Test through root mux ServeHTTP (tests routing!)
    // This tests: routing, method matching, handler execution
    mux.ServeHTTP(rec, req)
    
    // ❌ WRONG: handler.Create(rec, req) - bypasses routing layer!
    
    // Verify HTTP response
    if rec.Code != http.StatusCreated {
        t.Fatalf("Expected status %d, got %d", http.StatusCreated, rec.Code)
    }
    
    // Parse protobuf response
    var response pb.Product
    if err := json.NewDecoder(rec.Body).Decode(&response); err != nil {
        t.Fatalf("Failed to decode response: %v", err)
    }
    
    // Verify response using protocmp (Principle VI)
    expected := &pb.Product{
        Id:   response.Id,        // Generated UUID
        Name: reqData.Name,       // From request fixture
        Sku:  reqData.Sku,        // From request fixture
    }
    
    if diff := cmp.Diff(expected, &response, protocmp.Transform()); diff != "" {
        t.Errorf("Response mismatch (-want +got):\n%s", diff)
    }
    
    // Verify data committed to database
    var found Product
    if err := db.First(&found, response.Id).Error; err != nil {
        t.Fatalf("Product not found in database: %v", err)
    }
    
    // Truncate cleans up at end (even on test failure)
}

// Cleaner approach using helper function and mux.ServeHTTP
func TestProductCreateWithHelper(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products", "orders", "order_items")
    
    // Setup routes (shared configuration)
    service := NewProductService(db)
    mux := SetupRoutes(service)
    
    // Create HTTP request
    reqData := &pb.CreateProductRequest{
        Name: "Test Product",
        Sku:  "TEST-001",
    }
    body, _ := json.Marshal(reqData)
    req := httptest.NewRequest("POST", "/api/products", bytes.NewReader(body))
    rec := httptest.NewRecorder()
    
    // ✅ Test through root mux (tests routing)
    mux.ServeHTTP(rec, req)
    
    if rec.Code != http.StatusCreated {
        t.Fatalf("Expected status %d, got %d", http.StatusCreated, rec.Code)
    }
    
    // Parse and verify protobuf response
    var response pb.Product
    json.NewDecoder(rec.Body).Decode(&response)
    
    expected := &pb.Product{
        Id:   response.Id,   // Generated
        Name: reqData.Name,  // From fixture
        Sku:  reqData.Sku,   // From fixture
    }
    
    if diff := cmp.Diff(expected, &response, protocmp.Transform()); diff != "" {
        t.Errorf("Response mismatch (-want +got):\n%s", diff)
    }
}
```

**Parallel Test Support**:
```go
// Safe with testcontainers - each test gets own container
func TestProductCreateParallel(t *testing.T) {
    t.Parallel() // Each parallel test gets isolated container
    
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products", "orders", "order_items")
    
    // Setup routes (shared configuration)
    service := NewProductService(db)
    mux := SetupRoutes(service)
    
    // Create HTTP request
    reqData := &pb.CreateProductRequest{
        Name: "Test Product",
        Sku:  "TEST-001",
    }
    body, _ := json.Marshal(reqData)
    req := httptest.NewRequest("POST", "/api/products", bytes.NewReader(body))
    rec := httptest.NewRecorder()
    
    // ✅ Test through root mux (tests routing)
    mux.ServeHTTP(rec, req)
    
    if rec.Code != http.StatusCreated {
        t.Fatalf("Expected status %d, got %d", http.StatusCreated, rec.Code)
    }
    
    // Verify protobuf response
    var response pb.Product
    json.NewDecoder(rec.Body).Decode(&response)
    
    if response.Name != reqData.Name {
        t.Errorf("Expected name %s, got %s", reqData.Name, response.Name)
    }
}
```

**Internal GORM Model (Database Layer)**:
```go
// Internal model for database operations (can stay in internal/ package)
// Services use this internally but NEVER expose it in public APIs
type Product struct {
    ID   string `gorm:"primaryKey;type:uuid;default:gen_random_uuid()"`
    Name string `gorm:"not null"`
    SKU  string `gorm:"uniqueIndex;not null"`
}
```

**Service Implementation (Returns Protobuf)**:
```go
// Service interface - returns protobuf types (Principle XI)
type ProductService interface {
    Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error)
}

type productService struct {
    db *gorm.DB
}

func NewProductService(db *gorm.DB) *productServiceBuilder {
    return &productServiceBuilder{db: db}
}

func (b *productServiceBuilder) Build() ProductService {
    return &productService{
        db:     b.db,
        logger: b.logger,
        cache:  b.cache,
    }
}

func (s *productService) Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    // Normal production code - manages its own transaction
    tx := s.db.WithContext(ctx).Begin()
    defer tx.Rollback()
    
    // Use internal GORM model for database operations
    // Models are internal implementation details (Principle XI)
    product := &Product{Name: req.Name, SKU: req.Sku}
    
    if err := tx.Create(product).Error; err != nil {
        return nil, err
    }
    
    if err := tx.Commit().Error; err != nil {
        return nil, err
    }
    
    // ALWAYS return protobuf type (public API contract)
    // External apps only see protobuf types, never internal models
    return &pb.Product{
        Id:   product.ID,
        Name: product.Name,
        Sku:  product.SKU,
    }, nil
}
```

**HTTP Handler (Thin Wrapper)**:
```go
// Handler delegates to service (Principle XI)
type ProductHandler struct {
    service ProductService
}

func NewProductHandler(service ProductService) *ProductHandler {
    return &ProductHandler{service: service}
}

func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    var req pb.CreateProductRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }
    
    // Service returns protobuf type
    product, err := h.service.Create(ctx, &req)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(product)
}
```

**Routes Setup (Shared Between Production and Tests)**:
```go
// SetupRoutes creates the HTTP router with all routes registered
// CRITICAL: Production and tests MUST use the SAME routing configuration
// This ensures tests validate actual routing behavior
func SetupRoutes(service ProductService) http.Handler {
    mux := http.NewServeMux()
    handler := NewProductHandler(service)
    
    // Register routes using HTTP path patterns (Go 1.22+)
    // Pattern includes method and path parameters for type-safe routing
    mux.HandleFunc("POST /api/products", handler.Create)
    mux.HandleFunc("GET /api/products/{id}", handler.Get)
    mux.HandleFunc("PUT /api/products/{id}", handler.Update)
    mux.HandleFunc("DELETE /api/products/{id}", handler.Delete)
    
    return mux
}

// Example handler using path parameter extraction
func (h *ProductHandler) Get(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    // Extract path parameter using r.PathValue() (Go 1.22+)
    // NEVER use string manipulation of r.URL.Path
    id := r.PathValue("id")
    if id == "" {
        http.Error(w, "Missing product ID", http.StatusBadRequest)
        return
    }
    
    product, err := h.service.Get(ctx, id)
    if err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(product)
}
```


### Complete Testing Example (All Principles)

Here's a complete example that demonstrates all testing principles working together:

```go
// Complete test following ALL constitutional principles
func TestProductAPI_Create_Success(t *testing.T) {
    // Principle I: Integration test with real database
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    // Setup routes (Principle XI: Service Architecture)
    service := NewProductService(db)
    mux := SetupRoutes(service)  // ✅ Shared routing configuration
    
    // Principle II: Table-driven test design
    testCases := []struct {
        name     string
        request  *pb.CreateProductRequest
        expected func(*pb.Product) *pb.Product  // Function to build expected
    }{
        {
            name: "Create product with all fields",
            request: &pb.CreateProductRequest{
                Name: "Laptop",
                Sku:  "LAPTOP-001",
            },
            expected: func(resp *pb.Product) *pb.Product {
                // Principle VI: Build expected from FIXTURES, not response
                return &pb.Product{
                    Id:   resp.Id,        // Generated UUID (exception)
                    Name: "Laptop",       // From request fixture
                    Sku:  "LAPTOP-001",   // From request fixture
                }
            },
        },
        // Principle III: Edge cases
        {
            name: "Create product with minimum required fields",
            request: &pb.CreateProductRequest{
                Name: "Minimal",
                Sku:  "MIN-001",
            },
            expected: func(resp *pb.Product) *pb.Product {
                return &pb.Product{
                    Id:   resp.Id,
                    Name: "Minimal",
                    Sku:  "MIN-001",
                }
            },
        },
    }
    
    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            // Principle V: Test through root mux ServeHTTP (tests routing!)
            body, _ := json.Marshal(tc.request)
            req := httptest.NewRequest("POST", "/api/products", bytes.NewReader(body))
            rec := httptest.NewRecorder()
            
            // ✅ CORRECT: Call root mux ServeHTTP
            // This tests: URL routing, method matching, path patterns, handler execution
            mux.ServeHTTP(rec, req)
            
            // ❌ WRONG: handler.Create(rec, req) - bypasses routing!
            
            // Verify HTTP response
            if rec.Code != http.StatusCreated {
                t.Fatalf("Expected status %d, got %d", http.StatusCreated, rec.Code)
            }
            
            // Principle VI: Use protobuf structs with protocmp
            var response pb.Product
            if err := json.NewDecoder(rec.Body).Decode(&response); err != nil {
                t.Fatalf("Failed to decode response: %v", err)
            }
            
            expected := tc.expected(&response)
            if diff := cmp.Diff(expected, &response, protocmp.Transform()); diff != "" {
                t.Errorf("Response mismatch (-want +got):\n%s", diff)
            }
            
            // Principle IV: Verify real database fixture
            var dbProduct Product
            if err := db.First(&dbProduct, response.Id).Error; err != nil {
                t.Errorf("Product not persisted to database: %v", err)
            }
        })
    }
}

// Principle III: Edge case tests - validation errors
func TestProductAPI_Create_ValidationErrors(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    service := NewProductService(db)
    mux := SetupRoutes(service)  // ✅ Use shared routing
    
    testCases := []struct {
        name           string
        request        *pb.CreateProductRequest
        expectedStatus int
        expectedCode   string
    }{
        {
            name:           "Missing required name",
            request:        &pb.CreateProductRequest{Sku: "TEST-001"},
            expectedStatus: http.StatusBadRequest,
            expectedCode:   "MISSING_REQUIRED",
        },
        {
            name:           "Empty name",
            request:        &pb.CreateProductRequest{Name: "", Sku: "TEST-002"},
            expectedStatus: http.StatusBadRequest,
            expectedCode:   "MISSING_REQUIRED",
        },
        {
            name:           "Duplicate SKU",
            request:        &pb.CreateProductRequest{Name: "Product", Sku: "DUP-001"},
            expectedStatus: http.StatusConflict,
            expectedCode:   "DUPLICATE_SKU",
        },
    }
    
    // Create fixture for duplicate test (Principle IV: Real database fixtures)
    _ = createTestProduct(db, map[string]interface{}{
        "name": "Existing",
        "sku":  "DUP-001",
    })
    
    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            body, _ := json.Marshal(tc.request)
            req := httptest.NewRequest("POST", "/api/products", bytes.NewReader(body))
            rec := httptest.NewRecorder()
            
            // ✅ Test through root mux (tests routing)
            mux.ServeHTTP(rec, req)
            
            if rec.Code != tc.expectedStatus {
                t.Errorf("Expected status %d, got %d", tc.expectedStatus, rec.Code)
            }
            
            var errResp struct {
                Code    string `json:"code"`
                Message string `json:"message"`
            }
            json.NewDecoder(rec.Body).Decode(&errResp)
            
            if errResp.Code != tc.expectedCode {
                t.Errorf("Expected error code %s, got %s", tc.expectedCode, errResp.Code)
            }
        })
    }
}

// Helper function for creating test fixtures (Principle IV)
func createTestProduct(db *gorm.DB, data map[string]interface{}) *Product {
    product := &Product{
        Name: data["name"].(string),
        SKU:  data["sku"].(string),
    }
    db.Create(product)
    return product
}
```

**Key Takeaways**:
1. ✅ Integration test with real database (Principle I)
2. ✅ Table-driven design (Principle II)
3. ✅ Edge cases covered (Principle III)
4. ✅ Real database fixtures (Principle IV)
5. ✅ **Root mux ServeHTTP testing** - NOT individual handler methods (Principle V)
6. ✅ Protobuf with protocmp (Principle VI)
7. ✅ Service returns protobuf, models stay internal (Principle XI)
8. ✅ Expected values from fixtures, not response (Principle VI)
9. ✅ Shared routing configuration between production and tests (Principle V)
10. ✅ Path parameters tested via actual URL routing (Principle V)

---

### Code Review Requirements

All pull requests MUST be reviewed against these constitutional requirements:

**Principle I: Integration Testing First (No Mocking)**
- Pull requests MUST include integration tests for all new endpoints
- Reviewers MUST verify no mocking is used for database or HTTP layers
- Reviewers MUST verify tests use real PostgreSQL via testcontainers-go

**Principle II: Table-Driven Test Design**
- Reviewers MUST verify table-driven test structure (testCases := []struct)
- Reviewers MUST verify test cases have descriptive name fields
- Reviewers MUST verify shared setup/teardown extracted to helpers

**Principle III: Edge Case Coverage**
- Tests MUST demonstrate edge case coverage (validation, boundaries, errors, HTTP, security)
- Reviewers MUST verify SQL injection and XSS tests for text inputs
- Reviewers MUST verify negative values, zero values, and boundary conditions tested

**Principle IV: Real Database Fixtures**
- Reviewers MUST verify fixtures use GORM to insert into real database
- Reviewers MUST verify tests use database truncation for cleanup (defer pattern)
- Reviewers MUST verify truncation handles all tables modified by test
- Reviewers MUST verify truncation uses CASCADE for foreign key dependencies

**Principle V: ServeHTTP Endpoint Testing**
- Reviewers MUST verify tests call root mux `mux.ServeHTTP(rec, req)` (NOT individual handler methods)
- Reviewers MUST verify tests use httptest.NewRequest and httptest.NewRecorder
- Reviewers MUST verify tests exercise full HTTP stack including routing (not bypassing HTTP layer)
- Reviewers MUST verify production and tests use identical routing configuration (shared SetupRoutes function)
- Reviewers MUST verify path parameters are tested via actual URL paths (not mocked)
- Reviewers MUST reject tests that call handler methods directly (e.g., `handler.Create(rec, req)`)

**Principle VI: Protobuf Data Structures**
- Tests MUST use protobuf-generated structs (no `map[string]interface{}`)
- Tests MUST use `cmp.Diff()` with `protocmp.Transform()` for ALL protobuf message assertions
- Tests MUST NOT use individual field comparisons for protobuf messages
- Reviewers MUST verify `.proto` files are updated for API changes
- Reviewers MUST verify protobuf assertions use protocmp (not == or reflect.DeepEqual)

**Principle VII: Continuous Test Verification**
- Pull request MUST show evidence that AI agent ran tests and they passed
- Reviewers MUST reject PRs with disabled or skipped tests (unless justified in PR description)
- Reviewers MUST verify test execution time is reasonable (flag if tests take >5 minutes)
- Reviewers MUST verify flaky tests are fixed (not just re-run until they pass)
- Reviewers MUST verify AI agent ran tests after code changes (check commit messages/PR description)

**Principle VIII: Root Cause Tracing (Debugging Discipline)**
- Pull request MUST document root cause analysis for any bugs fixed
- Reviewers MUST verify fixes address root causes, not symptoms
- Reviewers MUST reject workarounds that mask underlying problems
- Reviewers MUST verify test cases were not removed or weakened to make tests pass
- Reviewers MUST verify test expectations reflect correct behavior, not broken behavior
- Reviewers MUST challenge "quick fixes" that lack root cause understanding
- PR description MUST include root cause trace for debugging work

**Principle IX: Acceptance Scenario Coverage (Spec-to-Test Mapping)**
- Reviewers MUST verify every acceptance scenario in spec.md has a corresponding test case
- Reviewers MUST verify table-driven test design is used for acceptance scenarios
- Reviewers MUST verify test case names reference source scenarios (US#-AS# in `name` field)
- Reviewers MUST reject implementations with untested acceptance scenarios (unless explicitly deferred with justification)
- Reviewers MUST verify tests validate complete "Then" clauses, not partial behavior
- PR description MUST include acceptance scenario coverage summary
- Reviewers SHOULD verify traceability matrix is up to date if provided (optional but recommended)

**Principle X: Test Coverage & Gap Analysis**
- Reviewers MUST verify that coverage reports were generated and analyzed
- Reviewers MUST verify that new code includes tests covering main paths and error conditions
- Reviewers MUST reject PRs that introduce significant coverage drops without justification
- Reviewers MUST challenge untested code paths identified by coverage analysis
