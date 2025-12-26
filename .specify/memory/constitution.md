<!--
## Sync Impact Report

**Version Change**: 3.2.0 → 3.3.0 (MINOR)
**Bump Rationale**: OgenHandler delegation pattern - services package contains OgenHandler that implements api.Handler and delegates to domain services.

**Modified Principles**:
- "API Data Structures (Schema-First Design)" - OgenHandler in services implements api.Handler, delegates to domain services
- "Service Layer Architecture" - OgenHandler delegation pattern, router builder creates ogen server
- "Error Handling" - ErrorResponse defined in OpenAPI, handlers package provides ErrorHandler

**Key Changes**:
- `services/ogen_handler.go` implements ogen `Handler` interface and delegates to domain services
- Domain services contain business logic with interfaces (e.g., `ProductService`)
- `handlers/routes.go` provides router builder that creates ogen server with ErrorHandler
- OgenHandler handles nil-service checks centrally
- External apps can import domain services or use OgenHandler for full API compatibility

**Templates Requiring Updates**:
- ✅ plan-template.md - Updated for OgenHandler pattern
- ✅ spec-template.md - Updated for OgenHandler pattern
- ✅ tasks-template.md - Updated for OgenHandler pattern
-->

# Go Project Constitution

This constitution defines the core principles and governance for Go microservice development. Detailed implementation guidelines, examples, and rationale are provided in separate appendix documents.


## Testing Principles

### INTEGRATION_TESTING. Integration Testing (No Mocking)

- Tests MUST use real PostgreSQL database connections (NO mocking)
- Test database MUST be isolated per test run with table truncation
- Fixtures MUST be inserted via GORM and represent realistic production data
- Implementation code MUST NEVER contain mocks or mock interfaces
- Dependencies requiring testability MUST be injected via interfaces using builder pattern (Principle SERVICE_ARCHITECTURE)
- Test code MUST NOT call `internal/` packages for setup - dependencies MUST be injected via service constructors
- **Exception (models-only fixtures)**: `testutil/` and fixture helpers MAY import `internal/models` strictly for inserting realistic database fixtures via GORM.
  - This exception applies ONLY to `internal/models` (NOT `internal/config`, NOT `internal/*` broadly)
  - Tests MUST still exercise behavior through public services/handlers (fixtures are setup only)
- Test setup MUST use public APIs and dependency injection (NOT direct internal package imports)
- Mocking is ONLY permitted in test code when testing interactions with external systems (third-party APIs, message queues)
- Mock implementations MUST be defined in `*_test.go` files (NEVER in production code files)
- Any use of mocks in tests MUST include written justification explaining why integration testing is infeasible

**Rationale**: Integration tests catch real-world issues mocks cannot (constraints, transactions, serialization). Real fixtures validate actual database behavior. Mocks in implementation code create fake abstraction layers that hide real behavior and complicate the codebase. Dependency injection provides testability without polluting production code with test doubles. Test code calling internal packages creates tight coupling and breaks encapsulation - all setup dependencies should flow through public service constructors using the builder pattern. When mocking is truly necessary (external services), it should be explicit, isolated to `*_test.go` files, and justified to prevent overuse.

**Test Package Organization**: Integration tests SHOULD be in a separate `tests/` package (NOT `services/*_test.go`). This keeps service packages clean and allows tests to import from multiple packages without circular dependencies. Test helpers (`setupTestDB`, `truncateTables`, fixture creators) go in `tests/testutil_test.go`.

### TABLE_DRIVEN. Table-Driven Design

- Test cases MUST be defined as slices of structs with descriptive `name` fields
- Execute using `t.Run(testCase.name, func(t *testing.T) {...})`

**Rationale**: Table-driven design reduces duplication and improves maintainability.

### EDGE_CASE_COVERAGE. Comprehensive Edge Case Coverage

Every endpoint MUST test:
- **Input validation**: Empty/nil values, invalid formats, SQL injection, XSS
- **Boundary conditions**: Zero/negative/max values, empty arrays, nil pointers
- **Auth**: Missing/expired/invalid tokens, insufficient permissions
- **Data state**: 404s, conflicts, concurrent modifications
- **Database**: Constraint violations, foreign key failures, transaction conflicts
- **HTTP**: Wrong methods, missing headers, invalid content-types, malformed JSON

**Rationale**: Edge case coverage prevents vulnerabilities and panics.

### SERVEHTTP_TESTING. ServeHTTP Endpoint Testing

API endpoints MUST be tested via ServeHTTP interface:
- Tests MUST call **root mux ServeHTTP** (NOT individual handler methods) using `httptest.ResponseRecorder`
- Tests MUST use **identical routing configuration** from shared routes package
- Tests MUST use **HTTP path patterns** (e.g., `"POST /api/v1/products/{productID}"`)
- Path parameters MUST be extracted using **`r.PathValue()`** (NOT string manipulation)
- Tests MUST verify response status codes, headers, and JSON body structure

**Rationale**: Testing through root mux ServeHTTP validates the complete HTTP stack (routing, middleware, parsing, error handling). Shared routing configuration ensures test accuracy. Path patterns provide type-safe parameter extraction.

**Why Root Mux (Not Individual Handlers)**:
Calling `handler.Create(rec, req)` bypasses routing, middleware, and method matching. Tests pass even with broken route registration (e.g., typo in path). Calling `mux.ServeHTTP(rec, req)` tests the complete stack and catches routing bugs before production.

### SCHEMA_FIRST. API Data Structures (Schema-First Design)

All public API data structures MUST use typed structs generated from OpenAPI specifications using **ogen** (github.com/ogen-go/ogen).

#### OpenAPI with ogen

Generate Go types and server interfaces directly from OpenAPI specifications:
- API contracts MUST be defined in OpenAPI specifications (single source of truth)
- Go code MUST be generated using `ogen` (github.com/ogen-go/ogen/cmd/ogen)
- Generated types provide native Go structs with JSON tags and built-in validation
- **CRITICAL**: `services/ogen_handler.go` MUST implement the generated `Handler` interface and delegate to domain services
- Domain services contain business logic with interfaces (e.g., `ProductService`)
- Services are reusable Go packages with OpenAPI-defined interfaces
- Tests MUST use generated structs (NO `map[string]interface{}`)
- **CRITICAL**: Tests MUST compare structs using `cmp.Diff()` (NO `==`, `reflect.DeepEqual`, or individual field checks)

**ogen Workflow**:
1. Define API contract in `api/openapi/<domain>.yaml` or `api/openapi/<domain>.json`
2. Generate Go code: `go run github.com/ogen-go/ogen/cmd/ogen@latest --target api/gen/<domain> --clean api/openapi/<domain>.yaml`
3. Implement the generated `Handler` interface in `services/` package
4. Use generated types in services and tests

**go:generate example**:
```go
package <domain>

//go:generate go run github.com/ogen-go/ogen/cmd/ogen@latest --target . --clean ../../openapi/<domain>.yaml
```

#### Requirements

- Tests MUST use generated structs (NO `map[string]interface{}`)
- **CRITICAL**: Tests MUST compare using `cmp.Diff()`
- **AI Agent Requirement**: ALWAYS use `cmp.Diff()` for struct assertions. NEVER use individual field comparisons like `if resp.Field != expected`
- **Expected values MUST be derived from TEST FIXTURES** (request data, database fixtures, config)
- **Copy from response ONLY for truly random fields**: UUIDs, timestamps, crypto-rand tokens
- **Copying non-random response fields defeats testing** (test always passes)

**Rationale**: Schema-first design provides type safety and ensures API contracts are the source of truth. Generated types eliminate manual struct maintenance and reduce bugs. `cmp.Diff()` ensures complete struct validation. ogen provides built-in validation based on OpenAPI schema.

**Example (OpenAPI with ogen)**:
```go
// ✅ CORRECT: Use cmp.Diff for struct comparison
expected := api.Product{
    Id:          response.Id,          // Random UUID (use response)
    Name:        "Test Product",       // From request fixture
    CategoryId:  category.ID,          // From database fixture
    CreatedAt:   response.CreatedAt,   // Timestamp (use response)
}
if diff := cmp.Diff(expected, response); diff != "" {
    t.Errorf("Mismatch (-want +got):\n%s", diff)
}

// ❌ WRONG: Individual field comparisons
if response.Name != "Test Product" {  // NEVER do this!
    t.Errorf("Expected name Test Product, got %s", response.Name)
}
```

**Truly Random Fields** (copy from response):
- Database UUIDs (`id`, `customer_id`)
- Timestamps (`created_at`, `updated_at`)
- Crypto-rand tokens (`membership_number`, `referral_code`)

**Everything Else** (use fixture values):
- Request payload data, database fixtures, config values, defaults, enums, constants

**AI Agent Requirement**: Read `testutil/fixtures.go` to find `CreateTestXxx()` defaults before writing assertions. Document fixture sources in comments.

### CONTINUOUS_TESTING. Continuous Test Verification

Tests MUST be executed after every code change:
- Tests MUST pass before any commit (run locally with testcontainers)
- Test failures MUST be fixed immediately (NO skipping/disabling tests)
- Test execution time MUST be optimized if impacting velocity

**Rationale**: Continuous testing catches regressions early when context is fresh and fixes are cheap. Without it, comprehensive test suites become worthless.

**AI Agent Requirements**:
- Run full test suite (`go test -v ./...`) after any code changes
- Run with race detector (`go test -v -race ./...`) for concurrency safety
- Treat test failures as blocking issues requiring immediate resolution

### ROOT_CAUSE. Root Cause Tracing (Debugging Discipline)

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

### ACCEPTANCE_COVERAGE. Acceptance Scenario Coverage (Spec-to-Test Mapping)

Every user scenario in specifications MUST have corresponding automated tests:
- Each acceptance scenario (US#-AS#) in spec.md MUST have a test case
- Test case names MUST reference source scenarios (e.g., "US1-AS1: New customer enrolls")
- Test functions MUST use table-driven design with test case structs (Principle TABLE_DRIVEN)
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

### COVERAGE_ANALYSIS. Test Coverage & Gap Analysis

**AI Agent Workflow to Recover Untested Code Paths**:

1. **Identify gaps**: Run `go test -coverprofile=coverage.out ./...` then `go tool cover -func=coverage.out` to list uncovered functions/lines
2. **Analyze gaps**: Read files with low coverage to understand untested branches (error paths, edge cases, conditionals)
3. **Write tests**: Add table-driven test cases covering the identified gaps (following Principles INTEGRATION_TESTING through ACCEPTANCE_COVERAGE)
4. **Verify**: Re-run coverage to confirm gaps are closed (target >80% for business logic)
5. **Clean dead code**: If code cannot be reached legitimately, remove it rather than exempting

**Rationale**: Coverage analysis reveals untested edge cases and error paths missed during initial development.


## Development Workflow

### Test-First Development (TDD)

1. **Design**: Define API contract (OpenAPI) → generate Go code with ogen
2. **Write Tests**: Create table-driven integration tests using generated structs (verify they fail)
3. **Implement**: Write minimal code to make tests pass
4. **Run Tests**: Execute full suite after implementation (Principle CONTINUOUS_TESTING)
5. **Refactor**: Improve code while keeping tests green, run tests after each change
6. **Complete**: Task done only when all tests pass

**Critical**: NO implementation before tests. Run tests after EVERY code change (Principle CONTINUOUS_TESTING).

### Bug Fix Flow (Reproduction-First Debugging)

When a bug is reported (e.g., via curl request/response, error logs, or user report), follow this systematic workflow:

**Phase 1: Capture & Analyze**
1. **Capture the failing request**: Save the exact curl command, request body, headers, and error response
2. **Identify the endpoint**: Extract HTTP method, path, and path parameters
3. **Extract test data**: Parse the JSON request body to create test fixtures using generated types (Principle SCHEMA_FIRST)
4. **Document expected vs actual**: Note what the response should be vs what was returned

**Phase 2: Reproduce with Integration Test**
1. **Write a failing test FIRST** (Principle INTEGRATION_TESTING: Integration Testing - NO mocking)
2. **Test through root mux ServeHTTP** (Principle SERVEHTTP_TESTING) - NOT individual handler methods
3. **Use table-driven design** (Principle TABLE_DRIVEN) with descriptive name including bug reference (e.g., `"BUG-123: PUT workflow returns INVALID_REQUEST for valid branch step"`)
4. **Use generated structs** (Principle SCHEMA_FIRST) for request/response - NO `map[string]interface{}`
5. **Setup realistic fixtures** via GORM representing the bug's preconditions
6. **Verify the test fails** with the same error as the reported bug

**Phase 3: Root Cause Analysis** (Principle ROOT_CAUSE)
1. **Trace backward**: Follow the call chain from error response to origin
2. **Distinguish symptoms from causes**: The error message is a symptom, find the root cause
3. **Use debugger/logging**: Add temporary logging to understand control flow
4. **Form hypotheses**: List potential causes and systematically eliminate them
5. **Document findings**: Record the root cause before implementing fix

**Phase 4: Fix & Verify**
1. **Fix at the source**: Address root cause, NOT symptoms (Principle ROOT_CAUSE)
2. **Run the reproduction test**: Verify it now passes
3. **Run full test suite**: Ensure no regressions (`go test -v -race ./...`) (Principle CONTINUOUS_TESTING)
4. **Update documentation**: If the bug revealed unclear API behavior, update docs


**Test Naming Convention for Bug Fixes**:
- Format: `"BUG-<ID>: <brief description of the bug>"`
- Example: `"BUG-123: PUT workflow returns INVALID_REQUEST for valid branch step"`
- Include bug ID in test case struct for traceability

**AI Agent Requirements**:
- AI agents MUST write a failing reproduction test BEFORE attempting any fix
- AI agents MUST NOT skip the reproduction step even for "obvious" bugs
- AI agents MUST verify the test fails with the reported error before proceeding
- AI agents MUST apply Root Cause Tracing (Principle ROOT_CAUSE) - no superficial fixes
- AI agents MUST run full test suite after fix to catch regressions (Principle CONTINUOUS_TESTING)

### OpenAPI with ogen Workflow

Generate Go types and server interfaces directly from OpenAPI specifications using `ogen`. **OgenHandler in services package implements the Handler interface and delegates to domain services**, making them reusable Go packages.

**Prerequisites**:
- `ogen` MUST be installed: `go install github.com/ogen-go/ogen/cmd/ogen@latest`

**Workflow**:
1. **Define Contract**: Create or update OpenAPI files in `api/openapi/<domain>.yaml` (YAML preferred) or `.json`
2. **Generate Go Code**: Run `go generate` to generate Go types and server interfaces
3. **Implement OgenHandler**: Create `services/ogen_handler.go` that implements `api.Handler` and delegates to domain services
4. **Implement Domain Services**: Create domain services with interfaces (e.g., `ProductService`) containing business logic
5. **Setup Router**: Use `handlers/routes.go` router builder to create ogen server with ErrorHandler
6. **Use in Code**: Import generated packages, use typed structs throughout
7. **Update**: Regenerate code whenever OpenAPI spec changes

**`go:generate` example**:
```go
package <domain>

//go:generate go run github.com/ogen-go/ogen/cmd/ogen@latest --target . --clean ../../openapi/<domain>.yaml
```

**OgenHandler Implementation** (implements ogen Handler interface, delegates to domain services):
```go
// services/ogen_handler.go
package services

import (
    "context"
    
    api "yourapp/api/gen/product"
)

// OgenHandler implements api.Handler and delegates to domain services
type OgenHandler struct {
    productService ProductService
}

// Ensure OgenHandler implements the generated Handler interface
var _ api.Handler = (*OgenHandler)(nil)

func NewOgenHandler() *OgenHandlerBuilder {
    return &OgenHandlerBuilder{}
}

func (b *OgenHandlerBuilder) WithProductService(svc ProductService) *OgenHandlerBuilder {
    b.productService = svc
    return b
}

func (b *OgenHandlerBuilder) Build() *OgenHandler {
    return &OgenHandler{productService: b.productService}
}

func (h *OgenHandler) CreateProduct(ctx context.Context, req *api.CreateProductReq) (*api.Product, error) {
    if h.productService == nil {
        return nil, ErrMissingRequired
    }
    return h.productService.Create(ctx, req)
}
```

**Domain Service Implementation** (business logic):
```go
// services/product_service.go
package services

type ProductService interface {
    Create(ctx context.Context, req *api.CreateProductReq) (*api.Product, error)
}

type productServiceImpl struct {
    db *gorm.DB
}

func NewProductService(db *gorm.DB) *productServiceBuilder {
    return &productServiceBuilder{db: db}
}

func (b *productServiceBuilder) Build() ProductService {
    return &productServiceImpl{db: b.db}
}

func (s *productServiceImpl) Create(ctx context.Context, req *api.CreateProductReq) (*api.Product, error) {
    product := &models.Product{Name: req.Name}
    if err := s.db.WithContext(ctx).Create(product).Error; err != nil {
        return nil, err
    }
    return &api.Product{ID: product.ID, Name: product.Name}, nil
}
```

**Router Setup** (via handlers package with ErrorHandler):
```go
// handlers/routes.go
package handlers

import api "yourapp/api/gen/product"

func NewRouter(productService services.ProductService) *routerBuilder {
    return &routerBuilder{productService: productService}
}

func (b *routerBuilder) Build() (http.Handler, error) {
    handler := services.NewOgenHandler().
        WithProductService(b.productService).
        Build()
    
    server, err := api.NewServer(handler, api.WithErrorHandler(OgenErrorHandler))
    if err != nil {
        return nil, err
    }
    
    var h http.Handler = server
    for _, mw := range b.middlewares {
        h = mw(h)
    }
    return h, nil
}
```

```go
// cmd/api/main.go
func main() {
    db := setupDatabase()
    
    if os.Getenv("HIDE_ERROR_DETAILS") == "true" {
        handlers.SetHideErrorDetails(true)
    }
    
    productService := services.NewProductService(db).Build()
    handler, err := handlers.NewRouter(productService).
        WithMiddlewares(handlers.DefaultMiddlewares()...).
        Build()
    if err != nil {
        log.Fatal(err)
    }
    http.ListenAndServe(":8080", handler)
}
```

**Use as Go Package** (external apps can import directly):
```go
// In another application
import "yourapp/services"

// Use domain service directly
productSvc := services.NewProductService(db).Build()
product, err := productSvc.Create(ctx, &api.CreateProductReq{Name: "Test"})

// Or use OgenHandler for full API compatibility
handler := services.NewOgenHandler().WithProductService(productSvc).Build()
product, err := handler.CreateProduct(ctx, &api.CreateProductReq{Name: "Test"})
```

**AI Agent Requirements**:
- AI agents MUST check if OpenAPI spec exists in `api/openapi/` before generating code
- AI agents MUST install `ogen` if not available: `go install github.com/ogen-go/ogen/cmd/ogen@latest`
- AI agents MUST regenerate code whenever OpenAPI files change
- AI agents MUST create `services/ogen_handler.go` that implements `api.Handler` and delegates to domain services
- AI agents MUST create domain services with interfaces (e.g., `ProductService`) and implementations
- AI agents MUST create `handlers/routes.go` with router builder that creates ogen server with ErrorHandler
- AI agents MUST use generated request/response types (NO manual struct definitions)

### Test Database Management

**AI Agent Requirements**:
- Use `testcontainers.PostgresContainer` for automatic PostgreSQL lifecycle
- Setup schema with GORM `AutoMigrate` (matches production)
- Cleanup with `defer container.Terminate(ctx)` pattern

**Prerequisite**: Docker must be available (local/CI environments)

### Test Isolation Strategy

Use **database truncation** for test isolation:
- Truncate tables after each test using `defer` pattern
- Use `TRUNCATE ... CASCADE` to handle foreign key constraints
- Truncate in reverse dependency order (children before parents)
- Centralize truncation logic in helper function

**Benefits**: Tests real production behavior with commits, works with any code structure, simple and fast (~1-5ms overhead).

### Architecture Overview for Testing

**Data Flow**: HTTP → Service (implements ogen Handler) → GORM Model → Database

**Layers**:
- **HTTP**: JSON serialization handled by ogen-generated server
- **Service**: Implements ogen-generated `Handler` interface with business logic (in `services/` package)
- **Model**: Internal GORM models (for database only)

**Testing Requirements**:
- Test via **ogen server ServeHTTP** to test complete HTTP stack
- Use generated types for assertions, internal models for DB verification only
- Services are reusable - can be tested directly or through HTTP

**Complete Testing Example**:
```go
import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "net/http/httptest"
    "testing"
    
    "github.com/google/go-cmp/cmp"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
    
    api "yourapp/api/gen/pim"  // ogen generated types
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

// Test example following all principles
func TestProductCreate(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    // ✅ Create service and server using ogen
    // Service implements ogen Handler interface
    service := services.NewProductService(db).Build()
    server, _ := api.NewServer(service)
    
    // ✅ Use generated structs for request
    reqData := api.CreateProductReq{Name: "Test Product", Sku: "TEST-001"}
    body, _ := json.Marshal(reqData)
    req := httptest.NewRequest("POST", "/api/products", bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    rec := httptest.NewRecorder()
    
    server.ServeHTTP(rec, req)  // ✅ Test through ogen server
    
    if rec.Code != http.StatusCreated {
        t.Fatalf("Expected %d, got %d", http.StatusCreated, rec.Code)
    }
    
    var response api.Product
    respBody, _ := io.ReadAll(rec.Body)
    json.Unmarshal(respBody, &response)
    
    // ✅ CORRECT: Build expected from fixtures, use cmp.Diff
    expected := api.Product{
        Id:   response.Id,    // Random UUID (copy from response)
        Name: reqData.Name,   // From request fixture
        Sku:  reqData.Sku,    // From request fixture
    }
    
    // ✅ ALWAYS use cmp.Diff for struct comparison
    if diff := cmp.Diff(expected, response); diff != "" {
        t.Errorf("Mismatch (-want +got):\n%s", diff)
    }
    
    // ❌ NEVER do individual field comparisons like this:
    // if response.Name != "Test Product" { t.Errorf(...) }
    // if !response.Valid { t.Errorf(...) }
}
```


## System Architecture

### SERVICE_ARCHITECTURE. Service Layer Architecture (Dependency Injection)

**Core Requirements**:
- **CRITICAL**: `services/ogen_handler.go` MUST implement ogen-generated `Handler` interface and delegate to domain services
- **CRITICAL**: Domain services MUST have interfaces (e.g., `ProductService`) with implementations containing business logic
- **CRITICAL**: `handlers/routes.go` MUST provide router builder that creates ogen server with ErrorHandler
- OgenHandler handles nil-service checks centrally
- Services are reusable Go packages with OpenAPI-defined interfaces
- Services MUST NOT depend on HTTP types (only `context.Context` allowed)
- **ALL validation MUST be in services** (ogen provides schema validation, services add business validation)
- Services MUST be in public packages (NOT `internal/`) for reusability
- External dependencies MUST be injected via builder pattern
- Services MUST use builder pattern: `NewService(db).WithLogger(log).Build()`
- `cmd/main.go` MUST use `handlers.NewRouter().Build()` (NOT `api.NewServer()` directly)
- If `cmd/main.go` needs functionality from `internal/`, that code MUST be promoted to a public service

**Architecture**: `HTTP (ogen server with ErrorHandler) → OgenHandler (delegates) → Domain Service → GORM Model → Database`

**Rationale**: OgenHandler in services package implements ogen Handler interface and delegates to domain services, providing clean separation between API contract and business logic. The `handlers/` package encapsulates router setup with ErrorHandler to map service errors to user-friendly HTTP responses. External apps can import domain services directly or use OgenHandler for full API compatibility. ogen handles HTTP routing, request parsing, and response serialization. Services in `internal/` cannot be imported externally (Go visibility rules).

### Service Method Parameters: Generated Structs Only

**CRITICAL**: Service methods (Handler interface implementations) MUST use ogen-generated structs for ALL parameters and return types. NO primitives, NO maps.

```go
// ✅ CORRECT: Service implements ogen-generated Handler interface
type ProductService struct {
    db *gorm.DB
}

// Ensure ProductService implements the generated Handler interface
var _ api.Handler = (*ProductService)(nil)

func (s *ProductService) CreateProduct(ctx context.Context, req *api.CreateProductReq) (*api.Product, error) {
    // Business logic here
}

func (s *ProductService) GetProduct(ctx context.Context, params api.GetProductParams) (*api.Product, error) {
    // Business logic here
}

// ❌ WRONG: Custom interfaces with primitives
type ProductService interface {
    Create(ctx context.Context, name, sku string) (*api.Product, error)  // NO primitives!
    Get(ctx context.Context, id string) (*api.Product, error)            // NO primitives!
}
```

**Why Generated Structs Only**:
- Built-in validation from OpenAPI schema
- Easy to add optional fields (no breaking changes)
- API documentation from OpenAPI spec
- Type safety



**Package Structure**:

```go
// ✅ CORRECT: Services implement ogen Handler, handlers provides server setup
github.com/theplant/myapp/
├── api/
│   ├── openapi/  // PUBLIC - OpenAPI specifications (single source of truth)
│   │   └── pim.yaml
│   └── gen/pim/  // PUBLIC - ogen generated types and server
│       ├── gen.go           // go:generate directive
│       ├── oas_handlers_gen.go
│       ├── oas_server_gen.go
│       └── oas_schemas_gen.go
├── services/          // PUBLIC - implements ogen Handler interface
│   ├── product_service.go   // Implements api.Handler
│   ├── errors.go            // Sentinel errors
│   └── migrations.go        // AutoMigrate() for external apps
├── handlers/          // PUBLIC - server setup with ErrorHandler
│   ├── server.go            // NewServer() wrapper with ErrorHandler
│   ├── error_handler.go     // ogen ErrorHandler implementation
│   └── error_codes.go       // Error code definitions mapping to services.Err*
├── tests/             // SEPARATE package for integration tests
│   ├── testutil_test.go     // Test helpers (setupTestDB, fixtures)
│   └── product_test.go      // Integration tests
├── internal/          // INTERNAL - implementation only
│   ├── models/        // ✅ OK (services return generated types, not models)
│   └── config/        // Not exposed
└── cmd/
    └── api/
        └── main.go

// ❌ WRONG: Handler interface implementation in handlers/ package
github.com/theplant/myapp/
├── handlers/
│   └── product_handler.go   // ❌ NEVER implement ogen Handler here!

// ❌ WRONG: Services in internal (cannot import)
github.com/theplant/myapp/
└── internal/
    └── services/      // ❌ External apps cannot import!
```

**Why Models Stay Internal**:
Services return ogen-generated types (public), use GORM models internally only. External apps never see models.

**When to Use `internal/`**:
- ✅ Models, config, database helpers
- ❌ Services, handlers, generated types (must be public for reusability)


**Internal GORM Model**:
```go
// Internal model for database operations (can stay in internal/ package)
// Services use this internally but NEVER expose it in public APIs
type Product struct {
    ID   string `gorm:"primaryKey;type:uuid;default:gen_random_uuid()"`
    Name string `gorm:"not null"`
    SKU  string `gorm:"uniqueIndex;not null"`
}
```

**Database Migrations**:
Export `AutoMigrate()` function so external apps can migrate without importing internal models:

```go
// services/migrations.go (PUBLIC)
func AutoMigrate(db *gorm.DB) error {
    return db.AutoMigrate(&models.Product{}, &models.Order{}, ...)
}

// External app usage
services.AutoMigrate(db)  // ✅ Migrates schema
svc := services.NewProductService(db).Build()
```

**Service Implementation with Builder Pattern**:

```go
// services/product_service.go
package services

import (
    "context"
    
    api "yourapp/api/gen/product"
    "yourapp/internal/models"
    "gorm.io/gorm"
)

// ProductService implements api.Handler interface
type ProductService struct {
    db     *gorm.DB
    logger Logger  // Optional
    cache  Cache   // Optional
}

// Ensure ProductService implements the generated Handler interface
var _ api.Handler = (*ProductService)(nil)

// Unexported builder
type productServiceBuilder struct {
    db     *gorm.DB
    logger Logger
    cache  Cache
}

// Constructor: REQUIRED params only
func NewProductService(db *gorm.DB) *productServiceBuilder {
    return &productServiceBuilder{db: db}
}

// Chainable optional params
func (b *productServiceBuilder) WithLogger(logger Logger) *productServiceBuilder {
    b.logger = logger
    return b
}

func (b *productServiceBuilder) WithCache(cache Cache) *productServiceBuilder {
    b.cache = cache
    return b
}

func (b *productServiceBuilder) Build() *ProductService {
    return &ProductService{db: b.db, logger: b.logger, cache: b.cache}
}

// Service method (implements ogen Handler interface)
func (s *ProductService) CreateProduct(ctx context.Context, req *api.CreateProductReq) (*api.Product, error) {
    // Note: ogen provides built-in validation from OpenAPI schema
    // Additional business validation can be added here
    
    product := &models.Product{Name: req.Name, SKU: req.Sku}
    
    // Principle CONTEXT_AWARE: Use context-aware database operations
    if err := s.db.WithContext(ctx).Create(product).Error; err != nil {
        // Principle ERROR_HANDLING: Wrap errors with context
        if isDuplicateKeyError(err) {
            return nil, fmt.Errorf("create product SKU %s: %w", req.Sku, ErrDuplicateSKU)
        }
        return nil, fmt.Errorf("create product: %w", err)
    }
    
    // Principle SCHEMA_FIRST: Return ogen-generated type
    return &api.Product{Id: product.ID, Name: product.Name, Sku: product.SKU}, nil
}
```

**Usage**:
```go
// Minimal (tests)
service := services.NewProductService(db).Build()

// Production (with logging)
service := services.NewProductService(db).WithLogger(log).WithCache(cache).Build()

// Create server via handlers package (includes ErrorHandler)
server, err := handlers.NewServer(service)
if err != nil {
    log.Fatal(err)
}
http.ListenAndServe(":8080", server)
```

**Use as Go Package** (external apps can import directly):
```go
// In another application
import "yourapp/services"

svc := services.NewProductService(db).Build()
product, err := svc.CreateProduct(ctx, &api.CreateProductReq{Name: "Test"})
```

**Note**: With ogen, the generated server handles HTTP routing, request parsing, and response serialization automatically. Services only implement business logic. The `handlers/` package provides `NewServer()` wrapper that configures ErrorHandler for proper error mapping.



## Distributed Tracing

### DISTRIBUTED_TRACING. Distributed Tracing (OpenTracing)

**Requirements**:
- HTTP endpoints MUST create OpenTracing spans with operation name (e.g., "POST /api/products")
- Service methods SHOULD create child spans (e.g., "ProductService.Create")
- Database operations: ONE span per transaction (NOT per SQL query - too much overhead)
- External calls (HTTP, gRPC) MUST propagate trace context
- Errors MUST set `span.SetTag("error", true)`
- Spans MUST include tags: `http.method`, `http.url`, `http.status_code`

**Tracing Granularity**:
- ✅ HTTP endpoints, service operations, database transactions
- ❌ Individual SQL queries (use database-specific tools instead)

**Setup**:
- Development/Tests: Use `opentracing.NoopTracer{}`
- Production: Configure from environment variables (Jaeger, Zipkin, Datadog, etc.)

**Rationale**: Provides observability for debugging latency, understanding request flows, and identifying bottlenecks. Tracing at service level (not per-query) keeps overhead low.

**Example**:
```go
// Service method with tracing (implements ogen Handler interface)
func (s *ProductService) CreateProduct(ctx context.Context, req *api.CreateProductReq) (*api.Product, error) {
    span, ctx := opentracing.StartSpanFromContext(ctx, "ProductService.CreateProduct")
    defer span.Finish()
    
    product := &models.Product{Name: req.Name, SKU: req.Sku}
    if err := s.db.WithContext(ctx).Create(product).Error; err != nil {  // Principle CONTEXT_AWARE
        span.SetTag("error", true)
        return nil, err
    }
    
    return &api.Product{Id: product.ID, Name: product.Name, Sku: product.SKU}, nil
}
```


## Context-Aware Operations

### CONTEXT_AWARE. Context-Aware Operations

**Requirements**:
- Service methods MUST accept `context.Context` as first parameter
- HTTP handlers MUST use `r.Context()`
- Database operations MUST use `db.WithContext(ctx)`
- External HTTP calls MUST use `http.NewRequestWithContext(ctx, ...)`
- Long-running operations MUST check context cancellation periodically
- Tests MUST verify context cancellation behavior

**Rationale**: Enables timeout handling, graceful cancellation, trace propagation, and prevents resource leaks when clients disconnect.

**Example**:
```go
// Service method - context as first parameter (ogen Handler interface)
func (s *ProductService) CreateProduct(ctx context.Context, req *api.CreateProductReq) (*api.Product, error) {
    // Principle CONTEXT_AWARE: Check cancellation before expensive operations
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }
    
    // Note: ogen provides built-in validation from OpenAPI schema
    // Additional business validation can be added here
    
    // Principle CONTEXT_AWARE: Use context-aware database operations
    tx := s.db.WithContext(ctx).Begin()
    defer tx.Rollback()
    
    product := &models.Product{Name: req.Name, SKU: req.Sku}
    if err := tx.Create(product).Error; err != nil {
        // Principle ERROR_HANDLING: Wrap errors with context
        return nil, fmt.Errorf("create product: %w", err)
    }
    
    if err := tx.Commit().Error; err != nil {
        return nil, fmt.Errorf("commit transaction: %w", err)
    }
    
    return &api.Product{Id: product.ID, Name: product.Name, Sku: product.SKU}, nil
}

// Long-running - check cancellation periodically
func (s *ProductService) BulkUpdate(ctx context.Context, req *api.BulkUpdateReq) error {
    for i, update := range req.Updates {
        if i%100 == 0 {
            select {
            case <-ctx.Done():
                return ctx.Err()
            default:
            }
        }
        // Process update...
    }
    return nil
}
```



## Error Handling Strategy

### ERROR_HANDLING. Comprehensive Error Handling

**Two-Layer Strategy**:
- **Service Layer**: Sentinel errors (package-level vars) with `fmt.Errorf("%w")` wrapping
- **HTTP Layer**: Singleton error code struct with automatic service error mapping
- **Environment-Aware Details**: Full error chain exposed in dev, hidden in production via `HIDE_ERROR_DETAILS=true`
- **Testing**: ALL errors (sentinel + HTTP codes) MUST have test cases

**Service Layer**:

```go
// services/errors.go - Define sentinel errors
var (
    ErrProductNotFound = errors.New("product not found")
    ErrDuplicateSKU    = errors.New("SKU already exists")
    ErrMissingRequired = errors.New("required field missing")
)
```

**HTTP Layer**:

**CRITICAL**: ErrorResponse MUST be defined in OpenAPI schema so it's generated by ogen:
```yaml
# api/openapi/<domain>.yaml
components:
  schemas:
    ErrorResponse:
      type: object
      required:
        - code
        - message
      properties:
        code:
          type: string
          description: Error code, e.g., "PRODUCT_NOT_FOUND"
        message:
          type: string
          description: Human-readable message
        details:
          type: string
          description: Full error chain (empty in production)
```

**ErrorHandler Implementation** (in handlers package):
```go
// handlers/error_handler.go - ogen ErrorHandler implementation
package handlers

import (
    "context"
    "encoding/json"
    "errors"
    "net/http"
    
    api "yourapp/api/gen/product"
    "yourapp/services"
)

// OgenErrorHandler implements ogenerrors.ErrorHandler for ogen servers
// It maps service sentinel errors to user-friendly HTTP responses
func OgenErrorHandler(ctx context.Context, w http.ResponseWriter, r *http.Request, err error) {
    errCode := mapServiceError(err)
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(errCode.HTTPStatus)
    
    // Use ogen-generated ErrorResponse type
    resp := api.ErrorResponse{
        Code:    errCode.Code,
        Message: errCode.Message,
    }
    
    // Include details in development only
    if !defaultErrorConfig.hideErrorDetails && err != nil {
        resp.Details.SetTo(err.Error())
    }
    
    json.NewEncoder(w).Encode(resp)
}

func mapServiceError(err error) ErrorCode {
    // Check context errors first
    if errors.Is(err, context.Canceled) {
        return Errors.RequestCancelled
    }
    if errors.Is(err, context.DeadlineExceeded) {
        return Errors.RequestTimeout
    }
    
    // Check service sentinel errors
    for _, errCode := range AllErrors() {
        if errCode.ServiceErr != nil && errors.Is(err, errCode.ServiceErr) {
            return errCode
        }
    }
    
    return Errors.InternalError
}
```

```go
// handlers/error_codes.go - Singleton with automatic mapping
type ErrorCode struct {
    Code       string
    Message    string
    HTTPStatus int
    ServiceErr error  // Maps to service sentinel error
}

var Errors = struct {
    MissingRequired  ErrorCode
    ProductNotFound  ErrorCode
    DuplicateSKU     ErrorCode
    RequestCancelled ErrorCode
    RequestTimeout   ErrorCode
    InternalError    ErrorCode
}{
    MissingRequired:  ErrorCode{"MISSING_REQUIRED", "Required field missing", 400, services.ErrMissingRequired},
    ProductNotFound:  ErrorCode{"PRODUCT_NOT_FOUND", "Product not found", 404, services.ErrProductNotFound},
    DuplicateSKU:     ErrorCode{"DUPLICATE_SKU", "SKU already exists", 409, services.ErrDuplicateSKU},
    RequestCancelled: ErrorCode{"REQUEST_CANCELLED", "Request cancelled", 499, context.Canceled},
    RequestTimeout:   ErrorCode{"REQUEST_TIMEOUT", "Request timeout", 504, context.DeadlineExceeded},
    InternalError:    ErrorCode{"INTERNAL_ERROR", "Internal server error", 500, nil},
}

// Environment-aware error details configuration
type errorResponseConfig struct {
    hideErrorDetails bool
}

var defaultErrorConfig = &errorResponseConfig{hideErrorDetails: false}

// SetHideErrorDetails configures whether to hide error details (call at startup)
func SetHideErrorDetails(hide bool) {
    defaultErrorConfig.hideErrorDetails = hide
}

// RespondWithError sends error response with optional details
// In dev (default): includes full error chain in details field
// In production (HIDE_ERROR_DETAILS=true): details field is empty
func RespondWithError(w http.ResponseWriter, errCode ErrorCode, err error) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(errCode.HTTPStatus)
    
    errResp := api.ErrorResponse{
        Code:    errCode.Code,
        Message: errCode.Message,
    }
    
    // Include full error chain unless hidden (for frontend/AI debugging)
    if !defaultErrorConfig.hideErrorDetails && err != nil {
        errResp.Details = err.Error()
    }
    
    data, _ := json.Marshal(errResp)
    w.Write(data)
}
```

**Handler - Validation and Error Wrapping**:
```go
// Handler method with validation (implements ogen Handler interface)
// Note: ogen provides built-in validation from OpenAPI schema
// Additional business validation can be added in handler methods
func (h *ProductHandler) GetProduct(ctx context.Context, params api.GetProductParams) (*api.Product, error) {
    var product Product
    if err := h.db.WithContext(ctx).First(&product, "id = ?", params.Id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("get product %s: %w", params.Id, ErrProductNotFound)
        }
        return nil, fmt.Errorf("query product %s: %w", params.Id, err)
    }
    return &api.Product{Id: product.ID, Name: product.Name, Sku: product.SKU}, nil
}
```

**Note**: With ogen, HTTP routing and request parsing are handled automatically by the generated server. Your handler methods implement the generated `Handler` interface directly.

**Startup Configuration**:
```go
// cmd/api/main.go
func main() {
    // Set error details visibility from environment
    if os.Getenv("HIDE_ERROR_DETAILS") == "true" {
        handlers.SetHideErrorDetails(true)
    }
    // ... rest of setup
}
```

**Example Responses**:
```json
// Development (default): full error chain visible for debugging
{
    "code": "PRODUCT_NOT_FOUND",
    "message": "Product not found",
    "details": "get product abc-123: product not found"
}

// Production (HIDE_ERROR_DETAILS=true): details hidden for security
{
    "code": "PRODUCT_NOT_FOUND",
    "message": "Product not found"
}
```

**Testing** (ALL errors MUST have test cases):
```go
func TestProductAPI_Create_DuplicateSKU(t *testing.T) {
    // Principle INTEGRATION_TESTING: Integration test with real database
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    // Principle INTEGRATION_TESTING: Create real database fixture
    createTestProduct(db, map[string]interface{}{"name": "Existing", "sku": "DUP-001"})
    
    // Create service and ogen server (service implements Handler)
    service := services.NewProductService(db).Build()
    server, _ := api.NewServer(service)
    
    // Principle SCHEMA_FIRST: Use generated structs
    reqData := api.CreateProductReq{Name: "New Product", Sku: "DUP-001"}
    body, _ := json.Marshal(reqData)
    req := httptest.NewRequest("POST", "/api/products", bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    rec := httptest.NewRecorder()
    
    // Principle SERVEHTTP_TESTING: Test through ogen server ServeHTTP
    server.ServeHTTP(rec, req)
    
    // Verify HTTP status
    if rec.Code != http.StatusConflict {
        t.Errorf("Expected 409, got %d", rec.Code)
    }
    
    // Use error code definitions (NOT literal strings)
    var errResp api.ErrorResponse
    respBody, _ := io.ReadAll(rec.Body)
    json.Unmarshal(respBody, &errResp)
    
    if errResp.Code != services.Errors.DuplicateSKU.Code {
        t.Errorf("Expected %s, got %s", services.Errors.DuplicateSKU.Code, errResp.Code)
    }
}
```

**Error Assertions in Tests**:
- Tests MUST use `ErrorCode` definitions from `services/errors.go` (NOT literal strings)
- Tests MUST validate both `Code` and `Message` fields against definitions
- Example: `if errResp.Code != Errors.WorkflowNotFound.Code` (NOT `"workflow not found"`)

**Rationale**: Error code definitions provide single source of truth. Tests stay in sync when messages change.

**Key Benefits**:
- ✅ No switch statements (automatic mapping via ServiceErr field)
- ✅ Type-safe with `errors.Is()`
- ✅ Wrapping creates error breadcrumb trail
- ✅ ALL errors must be tested
- ✅ Error assertions use definitions (maintainable, refactoring-safe)
- ✅ Environment-aware details for debugging (dev) vs security (production)

**AI Agent Requirements**:
- AI agents MUST read the `details` field when debugging API errors
- AI agents MUST NOT assume `details` is always present (production hides it)
- AI agents SHOULD suggest setting `HIDE_ERROR_DETAILS=true` for production deployments


## Technology Stack

- **Language**: Go 1.22+ (recommend latest stable)
- **Database**: PostgreSQL 15+ (with JSONB support)
- **HTTP Framework**: ogen-generated server (github.com/ogen-go/ogen)
- **Database Access**: GORM (gorm.io/gorm with gorm.io/driver/postgres)
- **Distributed Tracing**: OpenTracing (github.com/opentracing/opentracing-go)
- **OpenAPI Contract**: OpenAPI v3 (YAML preferred, JSON supported)
- **OpenAPI → Go Code**: ogen (github.com/ogen-go/ogen/cmd/ogen) - generates types, server, and client
- **Testing**: Standard library `testing` package with `httptest`
- **Test Comparison**: google/go-cmp for struct assertions
- **Test Database**: testcontainers-go with PostgreSQL module (automatic Docker container management)
- **Migration Tool**: GORM AutoMigrate (for development and testing)



## Code Review Requirements

All pull requests MUST be reviewed against these constitutional requirements:

**General Requirements**:
- Tests MUST be reviewed before implementation code (TDD workflow)
- Reviewers MUST verify GORM is used for database access (no raw SQL unless justified)
- ALL tests MUST pass before code review approval (no exceptions)
- Fixes MUST address root causes, not symptoms (Principle ROOT_CAUSE)

## Governance

### Amendment Process

1. Constitution changes MUST be proposed in writing with rationale
2. Changes MUST be reviewed by project lead or team
3. Version MUST be incremented per semantic versioning:
   - **MAJOR**: Backward incompatible principle changes (e.g., removing no-mocking rule, allowing map[string]interface{})
   - **MINOR**: New principles added or major expansions (e.g., adding ogen requirement, adding tracing requirement, adding comprehensive error handling, adding context-aware operations)
   - **PATCH**: Clarifications, examples, typo fixes
4. All dependent templates and documentation MUST be updated to reflect changes

### Compliance

- All pull requests MUST comply with these principles
- Constitution violations MUST be justified in PR description
- Complexity that violates simplicity principles MUST document "why needed" and "simpler alternatives rejected"
- When in doubt: integration test over unit test, real database over mock, table-driven over individual tests, generated structs over maps, Docker test database over local installation

### Version Control

This constitution is version-controlled alongside code and follows the same review process as code changes.

**Version**: 3.3.0 | **Ratified**: 2025-11-20 | **Last Amended**: 2025-12-26
