<!--
SYNC IMPACT REPORT
Version: 1.4.3 → 1.4.4
Modified Principles:
- Enhanced Principle V (ServeHTTP Endpoint Testing) - consolidated multiple enhancements into comprehensive HTTP routing and testing requirements including root routes handler testing, identical routing configuration between production and tests, HTTP path patterns, and PathValue parameter extraction
- Enhanced Principle VI (Protobuf Data Structures) - added strict definitions and anti-patterns section
Added Sections: None
Removed Sections: None
Templates Requiring Updates: None
Follow-up TODOs: None
-->
# Go Project Constitution Template

## Table of Contents

- [Testing & Quality Assurance](#testing--quality-assurance)
  - [I. Integration Testing First (No Mocking)](#i-integration-testing-first-no-mocking)
  - [II. Table-Driven Test Design](#ii-table-driven-test-design)
  - [III. Edge Case Coverage (NON-NEGOTIABLE)](#iii-edge-case-coverage-non-negotiable)
  - [IV. Real Database Fixtures](#iv-real-database-fixtures)
  - [V. ServeHTTP Endpoint Testing](#v-servehttp-endpoint-testing)
  - [VI. Protobuf Data Structures](#vi-protobuf-data-structures)
  - [VII. Continuous Test Verification](#vii-continuous-test-verification)
  - [VIII. Root Cause Tracing (Debugging Discipline)](#viii-root-cause-tracing-debugging-discipline)
  - [IX. Acceptance Scenario Coverage (Spec-to-Test Mapping)](#ix-acceptance-scenario-coverage-spec-to-test-mapping)
  - [X. Test Coverage & Gap Analysis](#x-test-coverage--gap-analysis)

- [System Architecture](#system-architecture)
  - [XI. Service Layer Architecture (Dependency Injection)](#xi-service-layer-architecture-dependency-injection)
  - [XII. Distributed Tracing (OpenTracing)](#xii-distributed-tracing-opentracing)

- [Operational Excellence](#operational-excellence)
  - [XIII. Comprehensive Error Handling](#xiii-comprehensive-error-handling)
  - [XIV. Context-Aware Operations](#xiv-context-aware-operations)

- [Technology Stack](#technology-stack)
- [Development Workflow](#development-workflow)
- [Governance](#governance)

## Testing & Quality Assurance

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
message ProductCreateRequest {
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
req := &pb.ProductCreateRequest{
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

## System Architecture

### XI. Service Layer Architecture (Dependency Injection)

Business logic MUST be separated from HTTP transport using service interfaces:
- Each HTTP endpoint handler MUST create or continue an OpenTracing span
- Spans MUST include operation name matching the endpoint (e.g., "POST /api/products")
- Service method calls SHOULD create child spans (e.g., "ProductService.Create")
- Database operations SHOULD be traced as a single child span per transaction (NOT per SQL query)
- Individual SQL queries MUST NOT be traced (too much overhead, use database metrics instead)
- External service calls (HTTP, gRPC) MUST propagate trace context and create spans
- Error conditions MUST be logged to the active span with `span.SetTag("error", true)`
- Trace context MUST be extracted from incoming HTTP headers (e.g., `X-B3-TraceId`)
- Trace context MUST be injected into outgoing HTTP requests
- Spans MUST include relevant tags: `http.method`, `http.url`, `http.status_code`, `service.method`
- Tests MUST verify tracing instrumentation (e.g., using mock tracer or test spans)

**Rationale**: Distributed tracing provides critical observability for debugging latency issues, understanding request flows across services, identifying bottlenecks, and correlating logs across distributed systems. OpenTracing offers a vendor-neutral API compatible with Jaeger, Zipkin, and other tracing backends. Tracing at service operation level (not individual SQL queries) keeps overhead low while providing actionable insights. For SQL query analysis, use database-specific tools like `pg_stat_statements`, slow query logs, or APM database profiling.

**Example - Correct Tracing Granularity**:
```go
import (
    "net/http"
    "github.com/opentracing/opentracing-go"
    "github.com/opentracing/opentracing-go/ext"
)

// HTTP Handler - creates root span
func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    // Extract or start HTTP span
    spanCtx, _ := opentracing.GlobalTracer().Extract(
        opentracing.HTTPHeaders,
        opentracing.HTTPHeadersCarrier(r.Header),
    )
    span := opentracing.StartSpan("POST /api/products", ext.RPCServerOption(spanCtx))
    defer span.Finish()
    
    span.SetTag("http.method", r.Method)
    span.SetTag("http.url", r.URL.String())
    
    // Parse request
    var req pb.ProductCreateRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        span.SetTag("error", true)
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }
    
    // Service call creates child span (one span for entire operation)
    ctx := opentracing.ContextWithSpan(r.Context(), span)
    product, err := h.service.Create(ctx, &req)
    
    if err != nil {
        span.SetTag("error", true)
        span.SetTag("error.message", err.Error())
        http.Error(w, err.Error(), 500)
        return
    }
    
    span.SetTag("http.status_code", http.StatusCreated)
    w.Header().Set("Content-Type", "application/json")
    if err := json.NewEncoder(w).Encode(product); err != nil {
        span.SetTag("error", true)
        // Log error (response already started, cannot change status)
        return
    }
}

// Service - creates child span for business logic
func (s *productService) Create(ctx context.Context, req *pb.ProductCreateRequest) (*pb.Product, error) {
    span, ctx := opentracing.StartSpanFromContext(ctx, "ProductService.Create")
    defer span.Finish()
    
    span.SetTag("product.name", req.Name)
    
    // Entire database transaction in this span
    // DO NOT create spans for individual SQL queries
    tx := s.db.WithContext(ctx).Begin()
    defer tx.Rollback()
    
    product := &Product{Name: req.Name, SKU: req.Sku}
    if err := tx.Create(product).Error; err != nil {
        span.SetTag("error", true)
        return nil, err
    }
    
    if err := tx.Commit().Error; err != nil {
        span.SetTag("error", true)
        return nil, err
    }
    
    return toProto(product), nil
}

// Result: Clean trace hierarchy
// POST /api/products (100ms)
//   └─ ProductService.Create (95ms)  ← Database work included here
//
// NOT this (too noisy):
// POST /api/products (100ms)
//   └─ ProductService.Create (95ms)
//       ├─ db.begin (1ms)
//       ├─ db.insert (50ms)
//       ├─ db.select (30ms)
//       └─ db.commit (10ms)  ← Too much detail!
```

**For SQL Query Analysis**: Use PostgreSQL's `pg_stat_statements` extension, slow query logs, or database monitoring tools (not distributed tracing).
- Business logic MUST be implemented as Go interfaces (service layer)
- Services MUST NOT depend on HTTP types (`http.Request`, `http.ResponseWriter`, `context.Context` is allowed)
- HTTP handlers MUST be thin wrappers that call service methods
- Services MUST accept all inputs as method parameters (no HTTP request parsing in services)
- External dependencies (database, logger, cache, etc.) MUST be injected via constructor
- Services MUST be exported and usable as normal Go packages
- Service interfaces MUST be defined in the same package as implementation
- Multiple applications MUST be able to share the same service instances
- **Services MUST NOT be in `internal/` packages** if they are intended for use by external Go applications
- Handlers MAY be in `internal/` packages if they are application-specific and not meant for external use
- Handlers CAN be in public packages if other applications want to embed or reuse them
- Use `internal/` ONLY for code that is truly internal implementation details, not part of the public API

**Rationale**: Separating business logic from HTTP transport enables code reuse across multiple contexts (HTTP APIs, gRPC services, CLI tools, background workers, embedded usage in other Go apps). Dependency injection allows applications to share expensive resources like database connection pools and caches. This architecture makes services testable without HTTP layer overhead and allows the package to be imported and used as a library in other Go applications. **CRITICAL**: Go's `internal/` package visibility rules prevent external applications from importing packages within `internal/`. Therefore, services must be in public packages (e.g., `services/`, `pkg/services/`) to enable cross-application reuse as described in the usage examples.

**Architecture Layers**:
```
HTTP Handler (thin) → Service Interface (business logic) → Repository (data access)
```

**Package Structure for Reusability**:

```go
// ✅ CORRECT: Services in public package (reusable)
github.com/yourorg/myapp/
├── services/          // PUBLIC - can be imported by other apps
│   ├── product_service.go
│   ├── order_service.go
│   └── errors.go      // Sentinel errors
├── handlers/          // PUBLIC - can be reused by other apps (optional)
│   ├── product_handler.go
│   └── error_codes.go
├── api/gen/pim/v1/    // PUBLIC - protobuf generated (MUST be importable)
│   ├── product.pb.go  // Returned by ProductService.Create()
│   └── template.pb.go
└── internal/          // INTERNAL - implementation details only
    ├── models/        // ✅ CAN be internal (services return protobuf, not models)
    │   └── product.go // Only used inside services for database mapping
    ├── database/      // Connection helpers (app-specific)
    ├── middleware/    // Logging, CORS (app-specific)
    └── config/        // Config loading (not exposed)

// ❌ WRONG: Services in internal (NOT reusable)
github.com/yourorg/myapp/
└── internal/
    ├── services/      // WRONG! Cannot be imported externally
    │   └── product_service.go
    └── handlers/
        └── product_handler.go

// External app trying to use services:
import "github.com/yourorg/myapp/internal/services"  // ❌ FAILS! Cannot import internal
```

**Why Models CAN Stay Internal** (Services Return Protobuf):
```go
// ✅ CORRECT: Services return protobuf (public), use models internally
// services/product_service.go (PUBLIC package)
package services

import (
    "yourapp/internal/models"  // ✅ OK! Internal use within service
    pb "yourapp/api/gen/pim/v1"  // ✅ PUBLIC protobuf types
    "gorm.io/gorm"
)

type ProductService interface {
    Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error)
    //                                                          ^^^^^^^^^^^^^
    //                                                          PUBLIC protobuf type!
}

type productService struct {
    db *gorm.DB
}

func (s *productService) Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    // Use internal models for database
    product := &models.Product{  // ✅ Internal model (private)
        Name: req.Name,
        SKU:  req.Sku,
    }
    
    s.db.Create(product)  // Save GORM model
    
    // Return protobuf (public type)
    return &pb.Product{  // ✅ Public protobuf
        Id:   product.ID,
        Name: product.Name,
        Sku:  product.SKU,
    }, nil
}

// External app using service:
import (
    "yourapp/services"  // ✅ Can import
    pb "yourapp/api/gen/pim/v1"  // ✅ Can import protobuf
)

svc := services.NewProductService(db)
product, err := svc.Create(ctx, &pb.CreateProductRequest{...})
if err != nil {
    log.Fatal(err)
}
// product is *pb.Product (public protobuf) - external app can use it! ✅
// models.Product is never exposed - stays internal ✅
```

**Key Insight**: Because services follow Principle VI (Protobuf Data Structures), they return protobuf types, not GORM models. Models are **only used internally** for database mapping and **never exposed** in the public API. Therefore, models CAN safely remain in `internal/models/`.

**When to Use internal/**:
- ✅ Database connection helpers (not part of public API)
- ✅ **Middleware** (application-specific - logging format, CORS policies, etc.)
- ✅ **Models** (internal data structures - services return protobuf, not models)
- ✅ Configuration loaders (implementation detail)
- ✅ Private utilities (not meant for external use)
- ❌ **Services** (MUST be public - promised reusability)
- ❌ **Protobuf generated code** (api/gen/ - needed by external apps to call services)
- ⚠️  **Handlers** (depends - public if reusable, internal if app-specific)

**IMPORTANT - Protobuf Return Types**: Services return **protobuf types** (`*pb.Product`, `*pb.ProductTemplate`), NOT GORM models. This means:
- ✅ Models CAN stay in `internal/models/` (only used internally by services)
- ✅ Protobuf types (`api/gen/pim/v1/`) MUST be importable (already generated in non-internal location)
- ✅ External apps only need to import services and protobuf packages, not models
- ✅ Go's compiler is happy - public services return public types (protobuf)

**CRITICAL - Database Migrations for External Apps**: External applications using services need to run database migrations, but cannot import internal models. **Solution**: Services MUST export an `AutoMigrate()` function that runs migrations on behalf of external apps:

```go
// services/migrations.go (PUBLIC package)
package services

import (
    "gorm.io/gorm"
    "yourapp/internal/models"  // Internal use within service package
)

// AutoMigrate runs all necessary database migrations for services
// External apps MUST call this before using services
func AutoMigrate(db *gorm.DB) error {
    return db.AutoMigrate(
        &models.ProductTemplate{},
        &models.Product{},
        &models.ProductPricing{},
        &models.ProductInventory{},
        &models.ProductVariant{},
        &models.VariantPricing{},
        &models.VariantInventory{},
        &models.MediaFile{},
    )
}

// External app usage:
import "yourapp/services"

db, err := gorm.Open(...)
if err != nil {
    log.Fatal(err)
}
if err := services.AutoMigrate(db); err != nil {  // ✅ Migrates schema
    log.Fatal(err)
}
svc := services.NewProductService(db)  // ✅ Ready to use
```

This approach:
- ✅ Keeps models internal (encapsulation)
- ✅ Allows external apps to migrate schema
- ✅ Services control their own schema requirements
- ✅ No need to expose GORM model internals

**Middleware Rationale**: Middleware is application-specific implementation (logging format, CORS policies, recovery behavior). External apps reusing your handlers should apply their own middleware. Keep middleware internal unless you're building a middleware library.

**Example Service Interface**:
```go
// Service interface - pure business logic, no HTTP dependencies
type ProductService interface {
    Create(ctx context.Context, req *pb.ProductCreateRequest) (*pb.Product, error)
    Get(ctx context.Context, id string) (*pb.Product, error)
    List(ctx context.Context, limit, offset int) ([]*pb.Product, error)
    Update(ctx context.Context, id string, req *pb.ProductUpdateRequest) (*pb.Product, error)
    Delete(ctx context.Context, id string) error
}

// Service implementation with dependency injection
type productService struct {
    db     *gorm.DB
    logger Logger
    cache  Cache
}

// Constructor with dependency injection
func NewProductService(db *gorm.DB, logger Logger, cache Cache) ProductService {
    return &productService{
        db:     db,
        logger: logger,
        cache:  cache,
    }
}

// Service method - pure business logic, no HTTP types
func (s *productService) Create(ctx context.Context, req *pb.ProductCreateRequest) (*pb.Product, error) {
    // Validate business rules
    if req.Name == "" {
        return nil, errors.New("product name required")
    }
    
    // Create entity
    product := &Product{
        Name: req.Name,
        SKU:  req.Sku,
        Description: req.Description,
    }
    
    // Persist with transaction
    tx := s.db.WithContext(ctx).Begin()
    defer tx.Rollback()
    
    if err := tx.Create(product).Error; err != nil {
        s.logger.Error("failed to create product", err)
        return nil, err
    }
    
    if err := tx.Commit().Error; err != nil {
        return nil, err
    }
    
    // Convert to protobuf response
    return &pb.Product{
        Id:          product.ID,
        Name:        product.Name,
        Sku:         product.SKU,
        Description: product.Description,
    }, nil
}
```

**Service Customization with Middleware Pattern**:

Services MAY expose middleware functions for external extensibility. Middleware provides full control: run code before/after operations, abort execution, or transform inputs/outputs.

**Middleware Pattern**:
```go
// CreateMiddleware wraps the Create operation, providing full control
// Parameters:
//   - ctx: Request context
//   - req: Creation request
//   - next: Function to call core Create logic (can be skipped)
// Returns: (*pb.Product, error)
// 
// Middleware can:
//   - Run code before core logic
//   - Decide whether to call next() or abort
//   - Run code after core logic
//   - Transform inputs or outputs
//   - Handle or wrap errors
type CreateMiddleware func(
    ctx context.Context,
    req *pb.ProductCreateRequest,
    next func(context.Context, *pb.ProductCreateRequest) (*pb.Product, error),
) (*pb.Product, error)

// Service struct with pre-built middleware handler
type productService struct {
    db            *gorm.DB
    createHandler func(context.Context, *pb.ProductCreateRequest) (*pb.Product, error)  // Pre-built chain
}

// serviceBuilder provides fluent API for service configuration
// Note: Unexported type - external apps don't need to know about it
type serviceBuilder struct {
    db                *gorm.DB
    createMiddlewares []CreateMiddleware
    logger            Logger
    cache             Cache
}

// NewProductService creates a builder with REQUIRED parameters only
// Returns unexported builder with exported methods for fluent API
// Use With* methods to add optional configuration, then call Build()
func NewProductService(db *gorm.DB) *serviceBuilder {
    return &serviceBuilder{db: db}
}

// WithCreateMiddleware adds middleware to Create operation (chainable)
func (b *serviceBuilder) WithCreateMiddleware(mw CreateMiddleware) *serviceBuilder {
    b.createMiddlewares = append(b.createMiddlewares, mw)
    return b  // Return builder for chaining
}

// WithLogger sets optional logger (chainable)
func (b *serviceBuilder) WithLogger(logger Logger) *serviceBuilder {
    b.logger = logger
    return b
}

// WithCache sets optional cache (chainable)
func (b *serviceBuilder) WithCache(cache Cache) *serviceBuilder {
    b.cache = cache
    return b
}

// Build constructs the service and builds middleware chain ONCE
func (b *serviceBuilder) Build() ProductService {
    svc := &productService{
        db:     b.db,
        logger: b.logger,
        cache:  b.cache,
    }
    
    // Build middleware chain once (not on every call!)
    handler := svc.coreCreate
    
    // Wrap with middleware (reverse order for correct execution)
    for i := len(b.createMiddlewares) - 1; i >= 0; i-- {
        mw := b.createMiddlewares[i]
        next := handler
        handler = func(ctx context.Context, req *pb.ProductCreateRequest) (*pb.Product, error) {
            return mw(ctx, req, next)
        }
    }
    
    svc.createHandler = handler
    return svc
}

// Core create logic (private)
func (s *productService) coreCreate(ctx context.Context, req *pb.ProductCreateRequest) (*pb.Product, error) {
    // Standard validation
    if req.Name == "" {
        return nil, fmt.Errorf("name: %w", ErrMissingRequired)
    }
    
    // Create in database
    product := &Product{Name: req.Name, SKU: req.Sku}
    if err := s.db.WithContext(ctx).Create(product).Error; err != nil {
        return nil, fmt.Errorf("create product: %w", err)
    }
    
    return toProto(product), nil
}

// Public Create method simply executes pre-built chain
func (s *productService) Create(ctx context.Context, req *pb.ProductCreateRequest) (*pb.Product, error) {
    // Execute pre-built middleware chain (built once in constructor)
    return s.createHandler(ctx, req)
}
```

**External App Using Middleware**:
```go
import "yourapp/services"

func main() {
    db, err := gorm.Open(...)
    if err != nil {
        log.Fatal(err)
    }
    
    // Middleware 1: Validation
    validateSKU := func(
        ctx context.Context,
        req *pb.ProductCreateRequest,
        next func(context.Context, *pb.ProductCreateRequest) (*pb.Product, error),
    ) (*pb.Product, error) {
        // Run before core logic
        if !strings.HasPrefix(req.Sku, "ACME-") {
            return nil, errors.New("SKU must start with ACME-")
        }
        
        // Call next middleware or core logic
        return next(ctx, req)
    }
    
    // Middleware 2: Notification
    notifySlack := func(
        ctx context.Context,
        req *pb.ProductCreateRequest,
        next func(context.Context, *pb.ProductCreateRequest) (*pb.Product, error),
    ) (*pb.Product, error) {
        // Call next first
        product, err := next(ctx, req)
        if err != nil {
            return nil, err
        }
        
        // Run after core logic
        sendSlackNotification(fmt.Sprintf("Created: %s", product.Name))
        
        return product, nil
    }
    
    // Middleware 3: Audit logging
    auditLog := func(
        ctx context.Context,
        req *pb.ProductCreateRequest,
        next func(context.Context, *pb.ProductCreateRequest) (*pb.Product, error),
    ) (*pb.Product, error) {
        // Run before
        startTime := time.Now()
        userID := getUserIDFromContext(ctx)
        
        // Call next
        product, err := next(ctx, req)
        
        // Run after (even if error)
        auditDB.Log(&AuditEntry{
            Action:   "product.create",
            UserID:   userID,
            Duration: time.Since(startTime),
            Success:  err == nil,
            Details:  fmt.Sprintf("SKU: %s", req.Sku),
        })
        
        return product, err
    }
    
    // Create service with fluent API (chainable configuration)
    productSvc := services.NewProductService(db).  // Required params only
        WithCreateMiddleware(validateSKU).         // Optional: validation
        WithCreateMiddleware(auditLog).            // Optional: audit
        WithCreateMiddleware(notifySlack).         // Optional: notification
        WithLogger(logger).                        // Optional: logger
        WithCache(cache).                          // Optional: cache
        Build()                                    // Builds middleware chain ONCE
    
    // Use normally - middleware chain executes automatically
    product, err := productSvc.Create(ctx, &pb.ProductCreateRequest{
        Name: "Widget",
        Sku:  "ACME-001",
    })
    
    // Execution flow:
    // 1. validateSKU runs → validates → calls next
    // 2. auditLog runs (before) → calls next
    // 3. notifySlack runs → calls next
    // 4. coreCreate runs → creates product
    // 5. notifySlack runs (after) → sends notification
    // 6. auditLog runs (after) → logs audit
    // 7. Return product
}
```

**Middleware Design Best Practices**:
- ✅ **Builder Pattern**: Constructor takes REQUIRED params, With* methods for OPTIONAL params
- ✅ **Fluent API**: With* methods return builder for chaining
- ✅ **Build once**: Build() method constructs service and middleware chain
- ✅ Export middleware function type with clear documentation
- ✅ Middleware receives `next` function to call core logic
- ✅ Middleware can decide whether to call `next` (abort if needed)
- ✅ Chain middleware in correct order (first added executes first)
- ✅ Document execution order clearly
- ✅ Pass protobuf types (never internal models)
- ✅ Provide example middleware for common use cases
- ❌ Don't expose internal service state
- ❌ Don't rebuild middleware chain on every method call (inefficient)

**Common Middleware Use Cases**:
- **Authorization**: Check permissions before allowing operation
- **Validation**: Custom business rules that can reject operations
- **Rate limiting**: Throttle operations per user/client
- **Caching**: Check cache before database, update after
- **Audit logging**: Track all operations with timing
- **Notifications**: Send alerts on specific operations
- **Transformations**: Modify requests or responses
- **Error handling**: Wrap or transform errors
- **Retry logic**: Retry failed operations
- **Circuit breakers**: Fail fast when downstream is down


**HTTP Handler (Thin Wrapper)**:
```go
// HTTP handler delegates to service
type ProductHandler struct {
    service ProductService // Injected dependency
}

func NewProductHandler(service ProductService) *ProductHandler {
    return &ProductHandler{service: service}
}

func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    // Extract trace span (HTTP concern)
    span, ctx := opentracing.StartSpanFromContext(r.Context(), "POST /api/products")
    defer span.Finish()
    
    // Parse HTTP request (HTTP concern)
    var req pb.ProductCreateRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request", 400)
        return
    }
    
    // Delegate to service (business logic)
    product, err := h.service.Create(ctx, &req)
    if err != nil {
        span.SetTag("error", true)
        http.Error(w, err.Error(), 500)
        return
    }
    
    // Format HTTP response (HTTP concern)
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(product)
}
```

**Usage in Other Go Applications**:
```go
// Application 1: HTTP API server
func main() {
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatal(err)
    }
    logger := NewLogger()
    cache := NewCache()
    
    // Create service with shared dependencies
    productService := NewProductService(db, logger, cache)
    
    // Use in HTTP handlers
    handler := NewProductHandler(productService)
    http.HandleFunc("/api/products", handler.Create)
    http.ListenAndServe(":8080", nil)
}

// Application 2: Background worker (shares same db/logger)
func main() {
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatal(err)
    }
    
    // Run migrations first (REQUIRED for external apps)
    if err := services.AutoMigrate(db); err != nil {
        log.Fatalf("Migration failed: %v", err)
    }
    
    logger := NewLogger()
    cache := NewCache()
    
    // Share same service instances!
    productService := NewProductService(db, logger, cache)
    
    // Use directly without HTTP layer
    ctx := context.Background()
    products, err := productService.List(ctx, 100, 0)
    if err != nil {
        log.Fatal(err)
    }
    for _, p := range products {
        // Process products...
    }
}

// Application 3: Embedded in larger application
import "github.com/yourorg/apidemo2/services"  // PUBLIC package (not internal/)

func processOrders() {
    // Run migrations once at app startup
    services.AutoMigrate(sharedDB)
    
    // Import and use services directly
    productSvc := services.NewProductService(sharedDB, sharedLogger, sharedCache)
    
    product, err := productSvc.Get(ctx, productID)
    // Use product in your business logic
}

// Note: Services in internal/services would FAIL:
// import "github.com/yourorg/apidemo2/internal/services"  // ❌ Cannot import internal
```

**Testing Through HTTP Layer (Full Stack)**:
```go
// HTTP integration tests cover full stack: HTTP → Service → Repository
func TestProductHandler_Create(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    // Setup service with real dependencies
    service := NewProductService(db)
    
    // Setup HTTP handler
    handler := NewProductHandler(service)
    
    // Create HTTP request
    reqBody := &pb.ProductCreateRequest{
        Name: "Test Product",
        Sku:  "TEST-001",
    }
    body, _ := json.Marshal(reqBody)  // Error handling omitted for test setup brevity
    req := httptest.NewRequest("POST", "/api/products", bytes.NewReader(body))
    rec := httptest.NewRecorder()
    
    // Test through HTTP layer (exercises Service → Repository)
    handler.Create(rec, req)
    
    // Assert HTTP response status
    if rec.Code != http.StatusCreated {
        t.Errorf("Expected %d, got %d", http.StatusCreated, rec.Code)
    }
    
    // Parse response
    var response pb.CreateProductResponse
    if err := json.NewDecoder(rec.Body).Decode(&response); err != nil {
        t.Fatalf("Failed to decode response: %v", err)
    }
    
    // Use protocmp to compare entire response (MANDATORY per Principle VI)
    expected := &pb.CreateProductResponse{
        Product: &pb.Product{
            Id:          response.Product.Id,  // Use generated ID
            Name:        reqBody.Name,
            Sku:         reqBody.Sku,
            CreatedAt:   response.Product.CreatedAt,  // Use generated timestamp
            UpdatedAt:   response.Product.UpdatedAt,
        },
    }
    
    if diff := cmp.Diff(expected, &response, protocmp.Transform()); diff != "" {
        t.Errorf("Response mismatch (-want +got):\n%s", diff)
    }
    
    // Note: This test covers HTTP parsing, service business logic, 
    // and repository database operations - full integration!
}
```

### XII. Distributed Tracing (OpenTracing)

All API endpoints MUST be instrumented with distributed tracing at appropriate granularity:
- Each HTTP endpoint handler MUST create or continue an OpenTracing span
- Spans MUST include operation name matching the endpoint (e.g., "POST /api/products")
- Service method calls SHOULD create child spans (e.g., "ProductService.Create")
- Database operations SHOULD be traced as a single child span per transaction (NOT per SQL query)
- Individual SQL queries MUST NOT be traced (too much overhead, use database metrics instead)
- External service calls (HTTP, gRPC) MUST propagate trace context and create spans
- Error conditions MUST be logged to the active span with `span.SetTag("error", true)`
- Trace context MUST be extracted from incoming HTTP headers (e.g., `X-B3-TraceId`)
- Trace context MUST be injected into outgoing HTTP requests
- Spans MUST include relevant tags: `http.method`, `http.url`, `http.status_code`, `service.method`
- Tests MUST verify tracing instrumentation (e.g., using mock tracer or test spans)

**Rationale**: Distributed tracing provides critical observability for debugging latency issues, understanding request flows across services, identifying bottlenecks, and correlating logs across distributed systems. OpenTracing offers a vendor-neutral API compatible with Jaeger, Zipkin, and other tracing backends. Tracing at service operation level (not individual SQL queries) keeps overhead low while providing actionable insights. For SQL query analysis, use database-specific tools like `pg_stat_statements`, slow query logs, or APM database profiling.

**Example - Correct Tracing Granularity**:
```go
import (
    "net/http"
    "github.com/opentracing/opentracing-go"
    "github.com/opentracing/opentracing-go/ext"
)

// HTTP Handler - creates root span
func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    // Extract or start HTTP span
    spanCtx, _ := opentracing.GlobalTracer().Extract(
        opentracing.HTTPHeaders,
        opentracing.HTTPHeadersCarrier(r.Header),
    )
    span := opentracing.StartSpan("POST /api/products", ext.RPCServerOption(spanCtx))
    defer span.Finish()
    
    span.SetTag("http.method", r.Method)
    span.SetTag("http.url", r.URL.String())
    
    // Parse request
    var req pb.ProductCreateRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        span.SetTag("error", true)
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }
    
    // Service call creates child span (one span for entire operation)
    product, err := h.service.Create(r.Context(), &req)
    if err != nil {
        span.SetTag("error", true)
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }
    
    span.SetTag("http.status_code", http.StatusCreated)
    span.SetTag("product.id", product.Id)
    
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(product)
}

// Service - creates child span
func (s *productService) Create(ctx context.Context, req *pb.ProductCreateRequest) (*pb.Product, error) {
    span, ctx := opentracing.StartSpanFromContext(ctx, "ProductService.Create")
    defer span.Finish()
    
    // Database operation - one span per transaction, not per query
    span, ctx := opentracing.StartSpanFromContext(ctx, "ProductService.Create.DB")
    defer span.Finish()
    
    // Single transaction with multiple operations
    return s.db.Transaction(func(tx *gorm.DB) error {
        product := &Product{...}
        if err := tx.Create(product).Error; err != nil {
            span.SetTag("error", true)
            return err
        }
        
        // More operations in same transaction...
        // NO individual spans for each INSERT/UPDATE/SELECT
        
        return nil
    })
}
```

**Tracing Granularity Guidelines**:
- ✅ HTTP endpoints: Root spans
- ✅ Service operations: Child spans  
- ✅ Database transactions: Child spans
- ❌ Individual SQL queries: Too granular, high overhead
- ❌ HTTP client calls within services: Should be child spans if cross-service

## Operational Excellence

### XIII. Comprehensive Error Handling

Error handling MUST use a two-layer strategy: sentinel errors for internal flow and singleton HTTP error codes for client responses:

**Service Layer (Internal Error Flow)**:
- Errors MUST be wrapped using `fmt.Errorf()` with the `%w` verb to preserve error chains
- Sentinel errors (package-level variables) MUST be defined for domain-specific error types
- Error messages MUST add contextual information at each layer (operation name, parameters, entity IDs)
- Errors MUST NEVER be swallowed without logging or returning them upward
- Service layer MUST wrap errors with business context before returning to handlers

**HTTP Layer (Client-Facing Responses)**:
- HTTP error codes and messages MUST be defined using singleton struct instances
- Error codes MUST NOT be hardcoded strings scattered throughout the codebase
- Error type checking MUST use `errors.Is()` and `errors.As()` (NOT string comparison)
- HTTP handlers MUST translate service errors to appropriate error codes
- HTTP responses MUST NOT expose internal error details or stack traces to clients
- The singleton MUST use struct literal initialization for compile-time safety
- Error definitions MUST be organized by category (validation, not found, conflict, internal)

**Testing**:
- Tests MUST verify error wrapping chains using `errors.Is()` and `errors.As()`
- Tests MUST verify HTTP handlers return correct error codes for each error type
- **ALL defined sentinel errors MUST have corresponding test cases** (each ErrXxx must be tested)
- **ALL HTTP error codes MUST have corresponding test cases** (each Errors.Xxx must be tested)
- Tests MUST cover both success and error paths for every operation
- Error test cases MUST verify the complete error flow: Service (sentinel) → Handler (HTTP code) → Client (response)

**Rationale**: This two-layer approach provides type-safe error handling at both service and HTTP layers. Sentinel errors enable type-safe checking with `errors.Is()` across wrapped errors, while the HTTP singleton provides consistent client-facing error codes. Error wrapping creates an "error breadcrumb trail" for debugging without stack traces (Go standard library doesn't provide stack traces). The singleton pattern prevents hardcoded strings and provides IDE autocomplete. Together, these create a robust error handling strategy that's type-safe, debuggable, and maintains clean separation between internal and external concerns.

**CRITICAL - Error Testing**: Every defined error (both sentinel errors in services and HTTP error codes in handlers) MUST have at least one test case that triggers and verifies it. Untested error paths are bugs waiting to happen in production. Tests MUST verify the complete error flow from service (sentinel error) through handler (HTTP code mapping) to client (HTTP response).

**Complete Error Handling Example**:

**Step 1: Define Sentinel Errors (Service Layer)**:
```go
// services/errors.go
package services

import "errors"

// Domain-specific sentinel errors for type-safe error checking
var (
    ErrProductNotFound  = errors.New("product not found")
    ErrDuplicateSKU     = errors.New("SKU already exists")
    ErrInvalidSKU       = errors.New("invalid SKU format")
    ErrMissingRequired  = errors.New("required field missing")
    ErrValueOutOfRange  = errors.New("value out of range")
)
```

**Step 2: Define HTTP Error Codes with Service Error Mapping (Handler Layer)**:
```go
// handlers/error_codes.go
package handlers

import (
    "net/http"
    "github.com/yourapp/services"
)

// ErrorCode for HTTP responses with optional service error mapping
type ErrorCode struct {
    Code       string
    Message    string
    HTTPStatus int
    ServiceErr error  // Optional: Maps to service sentinel error for automatic checking
}

// Singleton instance with all HTTP error definitions
var Errors = struct {
    // Validation errors (400)
    InvalidRequest   ErrorCode
    ValidationFailed ErrorCode
    MissingRequired  ErrorCode
    ValueOutOfRange  ErrorCode
    InvalidType      ErrorCode
    
    // Not found errors (404)
    ProductNotFound  ErrorCode
    TemplateNotFound ErrorCode
    
    // Conflict errors (409)
    DuplicateSKU     ErrorCode
    AlreadyExists    ErrorCode
    
    // Internal errors (500)
    InternalError    ErrorCode
}{
    // Validation errors - mapped to service errors
    InvalidRequest:   ErrorCode{"INVALID_REQUEST", "Invalid request body", http.StatusBadRequest, services.ErrInvalidRequest},
    MissingRequired:  ErrorCode{"MISSING_REQUIRED", "Required field missing", http.StatusBadRequest, services.ErrMissingRequired},
    ValueOutOfRange:  ErrorCode{"VALUE_OUT_OF_RANGE", "Value out of range", http.StatusBadRequest, services.ErrValueOutOfRange},
    InvalidType:      ErrorCode{"INVALID_TYPE", "Invalid type", http.StatusBadRequest, services.ErrInvalidType},
    
    // Not found errors - mapped to service errors
    ProductNotFound:  ErrorCode{"PRODUCT_NOT_FOUND", "Product not found", http.StatusNotFound, services.ErrProductNotFound},
    TemplateNotFound: ErrorCode{"TEMPLATE_NOT_FOUND", "Template not found", http.StatusNotFound, services.ErrTemplateNotFound},
    
    // Conflict errors - mapped to service errors
    DuplicateSKU:     ErrorCode{"DUPLICATE_SKU", "SKU already exists", http.StatusConflict, services.ErrDuplicateSKU},
    AlreadyExists:    ErrorCode{"ALREADY_EXISTS", "Resource already exists", http.StatusConflict, services.ErrAlreadyExists},
    
    // Internal errors - no mapping (catch-all)
    InternalError:    ErrorCode{"INTERNAL_ERROR", "Internal server error", http.StatusInternalServerError, nil},
}

// AllErrors returns a slice of all error codes for iteration
func AllErrors() []ErrorCode {
    return []ErrorCode{
        Errors.InvalidRequest,
        Errors.MissingRequired,
        Errors.ValueOutOfRange,
        Errors.InvalidType,
        Errors.ProductNotFound,
        Errors.TemplateNotFound,
        Errors.DuplicateSKU,
        Errors.AlreadyExists,
        Errors.InternalError,
    }
}
```

**Step 3: Service Layer - Wrap with Sentinel Errors**:
```go
// services/product_service.go
package services

import (
    "context"
    "errors"
    "fmt"
    "gorm.io/gorm"
)

func (s *productService) Get(ctx context.Context, id string) (*pb.Product, error) {
    var product Product
    
    if err := s.db.WithContext(ctx).First(&product, "id = ?", id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            // Wrap with sentinel error for type-safe checking
            return nil, fmt.Errorf("get product %s: %w", id, ErrProductNotFound)
        }
        // Wrap other errors with context
        return nil, fmt.Errorf("query product %s: %w", id, err)
    }
    
    return toProto(&product), nil
}

func (s *productService) Create(ctx context.Context, req *pb.ProductCreateRequest) (*pb.Product, error) {
    // Validation returns sentinel error
    if req.Name == "" {
        return nil, fmt.Errorf("product name: %w", ErrMissingRequired)
    }
    
    product := &Product{Name: req.Name, SKU: req.Sku}
    
    if err := s.db.WithContext(ctx).Create(product).Error; err != nil {
        if isDuplicateKeyError(err) {
            return nil, fmt.Errorf("create product SKU %s: %w", req.Sku, ErrDuplicateSKU)
        }
        return nil, fmt.Errorf("create product: %w", err)
    }
    
    return toProto(product), nil
}
```

**Step 4: Handler Layer - Simplified Error Mapping**:
```go
// handlers/product_handler.go
package handlers

import (
    "context"
    "errors"
)

func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    var req pb.CreateProductRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        RespondWithError(w, Errors.InvalidRequest)
        return
    }
    
    // Call service
    product, err := h.service.Create(ctx, &req)
    if err != nil {
        // Automatic error mapping - NO SWITCH NEEDED!
        HandleServiceError(w, err)
        return
    }
    
    json.NewEncoder(w).Encode(product)
}

// HandleServiceError automatically maps service errors to HTTP responses
func HandleServiceError(w http.ResponseWriter, err error) {
    // Check context errors first
    if errors.Is(err, context.Canceled) {
        http.Error(w, "Request cancelled", 499)
        return
    }
    if errors.Is(err, context.DeadlineExceeded) {
        http.Error(w, "Request timeout", 504)
        return
    }
    
    // Iterate through all error codes and check ServiceErr mapping
    for _, errCode := range AllErrors() {
        if errCode.ServiceErr != nil && errors.Is(err, errCode.ServiceErr) {
            RespondWithError(w, errCode)
            return
        }
    }
    
    // Default: Internal error (don't expose details)
    RespondWithError(w, Errors.InternalError)
}

// Helper function
func RespondWithError(w http.ResponseWriter, errCode ErrorCode) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(errCode.HTTPStatus)
    json.NewEncoder(w).Encode(map[string]string{
        "code":    errCode.Code,
        "message": errCode.Message,
    })
}
```

**Complete Error Flow (Simplified)**:

1. Service validates → Returns: fmt.Errorf("name: %w", ErrMissingRequired)
2. Handler receives error
3. HandleServiceError() iterates through AllErrors()
4. Finds: Errors.MissingRequired.ServiceErr == services.ErrMissingRequired ✅
5. Auto-maps: RespondWithError(w, Errors.MissingRequired)
6. Client receives: {"code": "MISSING_REQUIRED", "message": "...", "status": 400}

**Benefits of ServiceErr Field**:
- ✅ **No switch statement** - automatic error mapping via iteration
- ✅ **Single source of truth** - mapping defined once in ErrorCode definition
- ✅ **Easy to add errors** - just add to Errors struct with ServiceErr field
- ✅ **Type-safe** - still uses errors.Is() under the hood
- ✅ **Maintainable** - add new error by adding one line to Errors singleton
- ✅ **Flexible** - ServiceErr is optional (nil for HTTP-only errors)

**Error Testing Requirements**:
```go
// For every sentinel error in services/errors.go, there MUST be a test
// Example: Testing ErrDuplicateSKU
func TestProductHandler_Create_DuplicateSKU(t *testing.T) {
    // Setup - create product with SKU
    // Test - try to create another product with same SKU
    // Verify:
    // 1. Service returns error with ErrDuplicateSKU wrapped
    // 2. Handler maps to Errors.DuplicateSKU
    // 3. Client receives 409 with DUPLICATE_SKU code
}

// For every HTTP error code, verify it can be triggered
// Untested errors = production bugs waiting to happen
```

**Why Two Layers (Simplified)**:
- **Service sentinel errors**: Domain logic errors (ErrProductNotFound) - ALL must be tested
- **HTTP error codes**: Client-facing codes with automatic mapping via ServiceErr field - ALL must be tested
- **No manual switch**: AllErrors() + ServiceErr field handles mapping
- **Clean handlers**: Just call HandleServiceError(w, err) - done!

### XIV. Context-Aware Operations

All I/O and long-running operations MUST accept and respect `context.Context`:
- All service methods MUST accept `context.Context` as the first parameter
- HTTP handlers MUST extract context from `*http.Request` using `r.Context()`
- Database operations MUST use context-aware GORM methods (e.g., `db.WithContext(ctx)`)
- External HTTP calls MUST propagate context for timeout and cancellation
- gRPC calls MUST propagate context for distributed tracing and cancellation
- Long-running operations MUST check context cancellation periodically
- Context MUST be used for OpenTracing span propagation (already covered in Principle VII)
- Context cancellation MUST be respected to prevent resource leaks
- Tests MUST verify context cancellation behavior for long-running operations
- Tests MUST use `context.WithTimeout()` or `context.WithCancel()` to simulate cancellation scenarios

**Rationale**: Context provides a standard way to propagate request-scoped values (like trace IDs), handle timeouts, and enable graceful cancellation across API boundaries. This prevents resource leaks from abandoned operations, enables coordinated shutdown, supports distributed tracing, and follows Go's idiomatic patterns for concurrent programming. Without context, operations cannot be cancelled and resources may leak when clients disconnect.

**Example - Service Layer with Context**:
```go
// Service interface - context as first parameter (MANDATORY)
type ProductService interface {
    Create(ctx context.Context, req *pb.ProductCreateRequest) (*pb.Product, error)
    Get(ctx context.Context, id string) (*pb.Product, error)
    List(ctx context.Context, limit, offset int) ([]*pb.Product, error)
    Update(ctx context.Context, id string, req *pb.ProductUpdateRequest) (*pb.Product, error)
    Delete(ctx context.Context, id string) error
}

// Service implementation respects context
func (s *productService) Create(ctx context.Context, req *pb.ProductCreateRequest) (*pb.Product, error) {
    // Start tracing span from context
    span, ctx := opentracing.StartSpanFromContext(ctx, "ProductService.Create")
    defer span.Finish()
    
    // Validate business rules
    if req.Name == "" {
        return nil, errors.New("product name required")
    }
    
    // Create entity
    product := &Product{
        Name:        req.Name,
        SKU:         req.Sku,
        Description: req.Description,
    }
    
    // Use context-aware database operations (MANDATORY)
    tx := s.db.WithContext(ctx).Begin()
    defer tx.Rollback()
    
    // Check context cancellation before expensive operations
    select {
    case <-ctx.Done():
        return nil, ctx.Err() // Return context error (timeout or cancellation)
    default:
        // Continue with operation
    }
    
    if err := tx.Create(product).Error; err != nil {
        s.logger.Error("failed to create product", err)
        return nil, err
    }
    
    if err := tx.Commit().Error; err != nil {
        return nil, err
    }
    
    return toProto(product), nil
}
```

**HTTP Handler - Extract Context**:
```go
// HTTP handler extracts context from request
func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    // Extract context from HTTP request (MANDATORY)
    ctx := r.Context()
    
    // Create tracing span with context
    span, ctx := opentracing.StartSpanFromContext(ctx, "POST /api/products")
    defer span.Finish()
    
    // Parse request
    var req pb.ProductCreateRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request", 400)
        return
    }
    
    // Pass context to service (context propagation)
    product, err := h.service.Create(ctx, &req)
    if err != nil {
        // Check if error was due to context cancellation
        if err == context.Canceled {
            http.Error(w, "Request cancelled", 499) // Client closed connection
            return
        }
        if err == context.DeadlineExceeded {
            http.Error(w, "Request timeout", 504) // Gateway timeout
            return
        }
        
        span.SetTag("error", true)
        http.Error(w, err.Error(), 500)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(product)
}
```

**External HTTP Calls with Context**:
```go
// Making HTTP calls to external services with context propagation
func (s *productService) FetchExternalData(ctx context.Context, url string) (*Data, error) {
    // Create HTTP request with context (MANDATORY)
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    
    // Inject tracing headers
    carrier := opentracing.HTTPHeadersCarrier(req.Header)
    if err := opentracing.GlobalTracer().Inject(
        opentracing.SpanFromContext(ctx).Context(),
        opentracing.HTTPHeaders,
        carrier,
    ); err != nil {
        return nil, err
    }
    
    // Make request - will respect context timeout/cancellation
    resp, err := s.httpClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    var data Data
    if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
        return nil, err
    }
    
    return &data, nil
}
```

**Testing Context Cancellation**:
```go
func TestProductService_Create_ContextCancellation(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    service := NewProductService(db, logger, cache)
    
    // Create context with immediate cancellation
    ctx, cancel := context.WithCancel(context.Background())
    cancel() // Cancel immediately
    
    req := &pb.ProductCreateRequest{
        Name: "Test Product",
        Sku:  "TEST-001",
    }
    
    // Service should respect cancellation
    product, err := service.Create(ctx, req)
    
    // Verify context cancellation was respected
    if err != context.Canceled {
        t.Errorf("Expected context.Canceled error, got: %v", err)
    }
    if product != nil {
        t.Error("Expected nil product when context cancelled")
    }
    
    // Verify no data was committed to database
    var count int64
    db.Model(&Product{}).Count(&count)
    if count != 0 {
        t.Error("Expected no products in database after cancellation")
    }
}

func TestProductService_Create_ContextTimeout(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    service := NewProductService(db, logger, cache)
    
    // Create context with very short timeout
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Nanosecond)
    defer cancel()
    
    time.Sleep(10 * time.Millisecond) // Ensure timeout expires
    
    req := &pb.ProductCreateRequest{
        Name: "Test Product",
        Sku:  "TEST-001",
    }
    
    // Service should respect timeout
    product, err := service.Create(ctx, req)
    
    // Verify timeout was respected
    if err != context.DeadlineExceeded {
        t.Errorf("Expected context.DeadlineExceeded error, got: %v", err)
    }
    if product != nil {
        t.Error("Expected nil product when context timeout")
    }
}
```

**Long-Running Operations**:
```go
// For operations that process many items, check context periodically
func (s *productService) BulkUpdate(ctx context.Context, updates []*pb.ProductUpdate) error {
    tx := s.db.WithContext(ctx).Begin()
    defer tx.Rollback()
    
    for i, update := range updates {
        // Check context cancellation every N iterations
        if i%100 == 0 {
            select {
            case <-ctx.Done():
                return ctx.Err() // Stop processing if cancelled
            default:
                // Continue processing
            }
        }
        
        // Perform update
        if err := tx.Model(&Product{}).
            Where("id = ?", update.Id).
            Updates(update).Error; err != nil {
            return err
        }
    }
    
    return tx.Commit().Error
}
```
## Technology Stack

- **Language**: Go 1.25+ (recommend latest stable)
- **Database**: PostgreSQL 15+ (with JSONB support)
- **HTTP Framework**: Standard library `net/http` using `http.ServeMux`
- **Database Access**: GORM (gorm.io/gorm with gorm.io/driver/postgres)
- **Distributed Tracing**: OpenTracing (github.com/opentracing/opentracing-go)
- **Protocol Buffers**: protoc compiler, protoc-gen-go, protoc-gen-go-grpc
- **Validation**: protoc-gen-validate for protobuf field validation
- **Testing**: Standard library `testing` package with `httptest`
- **Test Comparison**: google/go-cmp with protocmp for protobuf message assertions
- **Test Database**: testcontainers-go with PostgreSQL module (automatic Docker container management)
- **Migration Tool**: GORM AutoMigrate (for development and testing)

## Development Workflow

### Test-First Development (TDD)

1. **Design Phase**: Define API contract in `.proto` files (request/response messages)
2. **Generate Code**: Run `protoc` to generate Go structs from protobuf definitions
3. **Write Tests**: Create table-driven integration tests using protobuf structs (NOT maps)
4. **Verify Failure**: Run tests to confirm they fail (red phase)
5. **Review Tests**: Review test design with team/lead before implementation
6. **Implement**: Write minimal code to make tests pass (green phase)
7. **Run Tests**: Execute full test suite immediately after implementation (MANDATORY per Principle XI)
8. **Verify Success**: Confirm all tests pass (green phase)
9. **Refactor**: Improve code quality while keeping tests green
10. **Run Tests Again**: Execute tests after each refactoring change (MANDATORY per Principle XI)
11. **Complete Task**: Task is only complete when all tests pass
12. **No Implementation Before Tests**: Code written before test approval MUST be discarded

**Critical**: Steps 7, 8, 10, and 11 enforce Principle XI (Continuous Test Verification). AI agents MUST run tests after EVERY code change and verify they pass before task completion.

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

**Example Testcontainers Setup**:
```go
import (
    "context"
    "fmt"
    "testing"
    
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
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
    
    // Run GORM AutoMigrate
    if err := db.AutoMigrate(&Product{}, &Order{}); err != nil {
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

// Example test with inline truncation
func TestProductCreate(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    
    // Truncate tables after test (in reverse dependency order)
    defer func() {
        db.Exec("TRUNCATE TABLE order_items CASCADE")
        db.Exec("TRUNCATE TABLE orders CASCADE")
        db.Exec("TRUNCATE TABLE products CASCADE")
    }()
    
    // Handler uses normal code with real db and commits
    handler := NewProductHandler(db)
    product := &Product{Name: "Test Product", SKU: "TEST-001"}
    
    // Handler commits internally (normal production behavior)
    err := handler.Create(product)
    if err != nil {
        t.Fatalf("Failed to create product: %v", err)
    }
    
    // Verify committed data (tests actual behavior)
    var found Product
    if err := db.First(&found, product.ID).Error; err != nil {
        t.Fatalf("Product not found: %v", err)
    }
    
    // Truncate cleans up at end (even on test failure)
}

// Cleaner approach using helper function
func TestProductCreateWithHelper(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products", "orders", "order_items")
    
    handler := NewProductHandler(db)
    product := &Product{Name: "Test Product", SKU: "TEST-001"}
    
    if err := handler.Create(product); err != nil {
        t.Fatalf("Failed: %v", err)
    }
    
    // Verify product was created in database
    var found Product
    if err := db.First(&found, product.ID).Error; err != nil {
        t.Fatalf("Product not found: %v", err)
    }
    
    // Use simple field check for GORM model (not protobuf)
    if found.Name != product.Name {
        t.Errorf("Expected name %s, got %s", product.Name, found.Name)
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
    
    handler := NewProductHandler(db)
    product := &Product{Name: "Test", SKU: "TEST"}
    
    if err := handler.Create(product); err != nil {
        t.Fatalf("Failed: %v", err)
    }
    
    // Verify product exists
    var found Product
    db.First(&found, product.ID)
    // Assertions...
}
```

**Handler Implementation (Simple Production Code)**:
```go
// Normal handler with db dependency - no special test structure
type ProductHandler struct {
    db *gorm.DB
}

func NewProductHandler(db *gorm.DB) *ProductHandler {
    return &ProductHandler{db: db}
}

func (h *ProductHandler) Create(product *Product) error {
    // Normal production code - manages its own transaction
    tx := h.db.Begin()
    defer tx.Rollback()
    
    if err := tx.Create(product).Error; err != nil {
        return err
    }
    
    return tx.Commit().Error // Commits normally
}

// HTTP handler is straightforward
func (h *ProductHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    var req pb.ProductCreateRequest
    if err := json.NewDecoder(r.Body).Unmarshal(&req); err != nil {
        http.Error(w, "Invalid request", 400)
        return
    }
    
    product := &Product{Name: req.Name, SKU: req.Sku}
    if err := h.Create(product); err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    
    json.NewEncoder(w).Encode(product)
}
```

### Tracing Setup

- Development environment MUST use `opentracing.NoopTracer{}` (no external dependencies)
- Tests MUST use `opentracing.NoopTracer{}` by default
- Tests that verify tracing instrumentation MAY use mock tracer to validate spans
- Production MUST configure real tracing backend (deployment-specific: Jaeger, Zipkin, Datadog, etc.)
- Production tracing configuration MUST be loaded from environment variables

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
- Reviewers MUST verify tests use httptest.NewRequest and httptest.NewRecorder
- Reviewers MUST verify tests exercise full HTTP stack (not bypassing HTTP layer)

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

**Principle XI: Service Layer Architecture (Dependency Injection)**
- Reviewers MUST verify services are in public packages (not internal/)
- Reviewers MUST verify services do NOT depend on HTTP types (http.Request, http.ResponseWriter)
- Reviewers MUST verify handlers are thin wrappers delegating to services
- Reviewers MUST verify service methods accept only business parameters
- Reviewers MUST verify services use dependency injection (constructor parameters)
- Reviewers MUST verify AutoMigrate() is updated when models change
- Reviewers MUST verify services return protobuf types (not internal models)

**Principle XII: Distributed Tracing (OpenTracing)**
- Reviewers MUST verify OpenTracing spans are created for new endpoints
- Reviewers MUST verify spans include required tags (http.method, http.url, http.status_code)
- Reviewers MUST verify error spans are tagged with error=true

**Principle XIII: Comprehensive Error Handling**
- Reviewers MUST verify sentinel errors defined for domain-specific errors
- Reviewers MUST verify errors are wrapped with `fmt.Errorf("%w", err)` (NOT `%v`)
- Reviewers MUST verify error messages add contextual information at each layer
- Reviewers MUST verify error checking uses `errors.Is()` and `errors.As()` (NOT string comparison)
- Reviewers MUST verify errors are never swallowed without logging or returning
- Reviewers MUST verify HTTP error codes use singleton struct instances (no hardcoded strings)
- Reviewers MUST verify ErrorCode.ServiceErr field maps to service sentinel errors
- Reviewers MUST verify HandleServiceError uses AllErrors() iteration (no switch statement)
- Reviewers MUST verify HTTP handlers do NOT expose internal error details to clients
- Reviewers MUST verify tests use `errors.Is()` and `errors.As()` for error validation
- **Reviewers MUST verify ALL sentinel errors have test cases** (every ErrXxx tested)
- **Reviewers MUST verify ALL HTTP error codes have test cases** (every Errors.Xxx tested)
- **Reviewers MUST verify both success and error paths tested for every operation**

**Principle XIV: Context-Aware Operations**
- Reviewers MUST verify all service methods accept `context.Context` as first parameter
- Reviewers MUST verify HTTP handlers extract context from `r.Context()`
- Reviewers MUST verify database operations use `db.WithContext(ctx)`
- Reviewers MUST verify external HTTP calls use `http.NewRequestWithContext(ctx, ...)`
- Reviewers MUST verify context is propagated through all layers (HTTP → Service → Repository)
- Reviewers MUST verify context cancellation checks in Create/long-running operations
- Tests MUST verify context cancellation behavior where applicable

**General**
- Tests MUST be reviewed before implementation code (TDD workflow)
- Reviewers MUST verify GORM is used for database access (no raw SQL unless justified)
- ALL tests MUST pass before code review approval (no exceptions)
- Fixes MUST address root causes, not symptoms (Principle XII)

## Governance

### Amendment Process

1. Constitution changes MUST be proposed in writing with rationale
2. Changes MUST be reviewed by project lead or team
3. Version MUST be incremented per semantic versioning:
   - **MAJOR**: Backward incompatible principle changes (e.g., removing no-mocking rule, allowing map[string]interface{})
   - **MINOR**: New principles added or major expansions (e.g., adding protobuf requirement, adding tracing requirement, adding comprehensive error handling, adding context-aware operations)
   - **PATCH**: Clarifications, examples, typo fixes
4. All dependent templates and documentation MUST be updated to reflect changes

### Compliance

- All pull requests MUST comply with these principles
- Constitution violations MUST be justified in PR description
- Complexity that violates simplicity principles MUST document "why needed" and "simpler alternatives rejected"
- When in doubt: integration test over unit test, real database over mock, table-driven over individual tests, protobuf structs over maps, Docker test database over local installation

### Version Control

This constitution is version-controlled alongside code and follows the same review process as code changes.

**Version**: 1.4.4 | **Ratified**: 2025-11-20 | **Last Amended**: 2025-11-24

**Version History**:
- **1.4.4** (2025-11-24): Enhanced Principle V (ServeHTTP Endpoint Testing) - consolidated multiple enhancements into comprehensive HTTP routing and testing requirements including root routes handler testing, identical routing configuration between production and tests, HTTP path patterns, and PathValue parameter extraction; Enhanced Principle VI (Protobuf Data Structures) - added strict definitions and anti-patterns section
- **1.4.0** (2025-11-22): Added Principle XIV (Test Coverage & Gap Analysis) - requires using `go test -cover` to uncover and test untested code paths
- **1.3.3** (2025-11-22): Fixed example code violations - corrected all example code to follow Principle IX (Comprehensive Error Handling). Added proper error handling to examples in Principles VII and VIII where errors were being ignored. Added explanatory comments to test setup code where error handling omission is acceptable for brevity.
- **1.3.2** (2025-11-22): Enhanced Principle VI (Protobuf Data Structures) - added strict definitions and anti-patterns section. Explicitly defines what qualifies as "truly random/generated" (only UUIDs, timestamps, crypto/rand values). Prohibits claiming fixture default values are "unknowable" or "can't derive". Requires AI agents to read fixture code before writing tests. Adds zero-tolerance code review enforcement protocol.
- **1.3.1** (2025-11-21): Enhanced Principle VI (Protobuf Data Structures) - strengthened test fixture guidance to explicitly derive expected values from test fixtures (request data, DB fixtures, config) NOT response data, except for truly random/generated fields
- **1.3.0** (2025-11-21): Added Principle XIII (Acceptance Scenario Coverage) - requires one-to-one mapping between spec acceptance scenarios and integration tests using table-driven design and protocmp
- **1.2.0** (2025-11-20): Added Principle XII (Root Cause Tracing) - requires debugging problems at source, not symptoms
- **1.1.0** (2025-11-20): Added Principle XI (Continuous Test Verification) - requires running tests after all code changes
- **1.0.0** (2025-11-20): Initial release with 10 core principles for Go API development
