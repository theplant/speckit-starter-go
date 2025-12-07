
# Go Project Constitution

This constitution defines the core principles and governance for Go microservice development. Detailed implementation guidelines, examples, and rationale are provided in separate appendix documents.


## Testing Principles

### I. Integration Testing (No Mocking)

- Tests MUST use real PostgreSQL database connections (NO mocking)
- Test database MUST be isolated per test run with table truncation
- Fixtures MUST be inserted via GORM and represent realistic production data
- Implementation code MUST NEVER contain mocks or mock interfaces
- Dependencies requiring testability MUST be injected via interfaces using builder pattern (Principle X)
- Test code MUST NOT call `internal/` packages for setup - dependencies MUST be injected via service constructors
- Test setup MUST use public APIs and dependency injection (NOT direct internal package imports)
- Mocking is ONLY permitted in test code when testing interactions with external systems (third-party APIs, message queues)
- Mock implementations MUST be defined in `*_test.go` files (NEVER in production code files)
- Any use of mocks in tests MUST include written justification explaining why integration testing is infeasible

**Rationale**: Integration tests catch real-world issues mocks cannot (constraints, transactions, serialization). Real fixtures validate actual database behavior. Mocks in implementation code create fake abstraction layers that hide real behavior and complicate the codebase. Dependency injection provides testability without polluting production code with test doubles. Test code calling internal packages creates tight coupling and breaks encapsulation - all setup dependencies should flow through public service constructors using the builder pattern. When mocking is truly necessary (external services), it should be explicit, isolated to `*_test.go` files, and justified to prevent overuse.

### II. Table-Driven Design

- Test cases MUST be defined as slices of structs with descriptive `name` fields
- Execute using `t.Run(testCase.name, func(t *testing.T) {...})`

**Rationale**: Table-driven design reduces duplication and improves maintainability.

### III. Comprehensive Edge Case Coverage

Every endpoint MUST test:
- **Input validation**: Empty/nil values, invalid formats, SQL injection, XSS
- **Boundary conditions**: Zero/negative/max values, empty arrays, nil pointers
- **Auth**: Missing/expired/invalid tokens, insufficient permissions
- **Data state**: 404s, conflicts, concurrent modifications
- **Database**: Constraint violations, foreign key failures, transaction conflicts
- **HTTP**: Wrong methods, missing headers, invalid content-types, malformed JSON

**Rationale**: Edge case coverage prevents vulnerabilities and panics.

### IV. ServeHTTP Endpoint Testing

API endpoints MUST be tested via ServeHTTP interface:
- Tests MUST call **root mux ServeHTTP** (NOT individual handler methods) using `httptest.ResponseRecorder`
- Tests MUST use **identical routing configuration** from shared routes package
- Tests MUST use **HTTP path patterns** (e.g., `"POST /api/v1/products/{productID}"`)
- Path parameters MUST be extracted using **`r.PathValue()`** (NOT string manipulation)
- Tests MUST verify response status codes, headers, and JSON body structure

**Rationale**: Testing through root mux ServeHTTP validates the complete HTTP stack (routing, middleware, parsing, error handling). Shared routing configuration ensures test accuracy. Path patterns provide type-safe parameter extraction.

**Why Root Mux (Not Individual Handlers)**:
Calling `handler.Create(rec, req)` bypasses routing, middleware, and method matching. Tests pass even with broken route registration (e.g., typo in path). Calling `mux.ServeHTTP(rec, req)` tests the complete stack and catches routing bugs before production.

### V. Protobuf Data Structures

All public API data structures MUST be defined in Protocol Buffers:
- API contracts MUST be defined in `.proto` files (single source of truth)
- Tests MUST use protobuf structs (NO `map[string]interface{}`)
- **CRITICAL**: Tests MUST compare protobuf messages using `cmp.Diff()` with `protocmp.Transform()` (NO `==`, `reflect.DeepEqual`, or individual field checks)
- **AI Agent Requirement**: ALWAYS use `cmp.Diff(expected, got, protocmp.Transform())` for protobuf assertions. NEVER use individual field comparisons like `if resp.Field != expected`
- **Expected values MUST be derived from TEST FIXTURES** (request data, database fixtures, config)
- **Copy from response ONLY for truly random fields**: UUIDs, timestamps, crypto-rand tokens
- **Copying non-random response fields defeats testing** (test always passes)

**Rationale**: Protobuf provides type safety and schema-first design. Protocmp ensures complete message validation. Deriving expected values from fixtures (not responses) ensures tests actually validate correctness.

**Example**:
```go
// ✅ CORRECT: Use cmp.Diff with protocmp.Transform()
expected := &pb.Product{
    Id:          response.Id,          // Random UUID (use response)
    Name:        "Test Product",       // From request fixture
    CategoryId:  category.ID,          // From database fixture
    CreatedAt:   response.CreatedAt,   // Timestamp (use response)
}
if diff := cmp.Diff(expected, &response, protocmp.Transform()); diff != "" {
    t.Errorf("Mismatch (-want +got):\n%s", diff)
}

// ❌ WRONG: Individual field comparisons
if response.Name != "Test Product" {  // NEVER do this!
    t.Errorf("Expected name Test Product, got %s", response.Name)
}
if !response.Valid {  // NEVER do this!
    t.Errorf("Expected valid=true")
}

// ❌ WRONG: Copying non-random fields from response
expected := &pb.Product{
    Name: response.Name,  // Wrong! Test always passes
}
```

**Truly Random Fields** (copy from response):
- Database UUIDs (`id`, `customer_id`)
- Timestamps (`created_at`, `updated_at`)
- Crypto-rand tokens (`membership_number`, `referral_code`)

**Everything Else** (use fixture values):
- Request payload data, database fixtures, config values, defaults, enums, constants

**AI Agent Requirement**: Read `testutil/fixtures.go` to find `CreateTestXxx()` defaults before writing assertions. Document fixture sources in comments.

### VI. Continuous Test Verification

Tests MUST be executed after every code change:
- Tests MUST pass before any commit (run locally with testcontainers)
- Test failures MUST be fixed immediately (NO skipping/disabling tests)
- Test execution time MUST be optimized if impacting velocity

**Rationale**: Continuous testing catches regressions early when context is fresh and fixes are cheap. Without it, comprehensive test suites become worthless.

**AI Agent Requirements**:
- Run full test suite (`go test -v ./...`) after any code changes
- Run with race detector (`go test -v -race ./...`) for concurrency safety
- Treat test failures as blocking issues requiring immediate resolution

### VII. Root Cause Tracing (Debugging Discipline)

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

### VIII. Acceptance Scenario Coverage (Spec-to-Test Mapping)

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

### IX. Test Coverage & Gap Analysis

**AI Agent Workflow to Recover Untested Code Paths**:

1. **Identify gaps**: Run `go test -coverprofile=coverage.out ./...` then `go tool cover -func=coverage.out` to list uncovered functions/lines
2. **Analyze gaps**: Read files with low coverage to understand untested branches (error paths, edge cases, conditionals)
3. **Write tests**: Add table-driven test cases covering the identified gaps (following Principles I-VIII)
4. **Verify**: Re-run coverage to confirm gaps are closed (target >80% for business logic)
5. **Clean dead code**: If code cannot be reached legitimately, remove it rather than exempting

**Rationale**: Coverage analysis reveals untested edge cases and error paths missed during initial development.


## Development Workflow

### Test-First Development (TDD)

1. **Design**: Define API contract in `.proto` files → generate code with `protoc`
2. **Write Tests**: Create table-driven integration tests using protobuf structs (verify they fail)
3. **Implement**: Write minimal code to make tests pass
4. **Run Tests**: Execute full suite after implementation (Principle VI)
5. **Refactor**: Improve code while keeping tests green, run tests after each change
6. **Complete**: Task done only when all tests pass

**Critical**: NO implementation before tests. Run tests after EVERY code change (Principle VI).

### Protobuf Workflow

1. **Define Schema**: Create or update `.proto` files in `api/` directory
2. **Generate Code**: Run `go generate` to create Go structs (add `//go:generate` directives)
3. **Use in Code**: Import generated packages, use typed structs throughout
4. **Validate**: Implement validation in service layer (check required fields, formats, constraints)
5. **Version**: Use protobuf field numbers consistently (never reuse deleted field numbers)

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

**Data Flow**: HTTP → Handler → Service (protobuf) → GORM Model → Database

**Layers**:
- **HTTP**: JSON serialization of protobuf structs using `protojson` (camelCase)
- **Handler**: Thin wrapper, delegates to service, uses `DecodeProtoJSON`/`RespondWithProto` helpers
- **Service**: Business logic, accepts/returns protobuf (NEVER internal models)
- **Model**: Internal GORM models (for database only)

**Testing Requirements**:
- Test via **root mux ServeHTTP** (NOT individual handlers) to test routing/middleware
- Use protobuf for assertions, internal models for DB verification only
- Share identical routing configuration (SetupRoutes) between production and tests

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
    "google.golang.org/protobuf/encoding/protojson"
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

// Test example following all principles
func TestProductCreate(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    // ✅ Use builder pattern for services and routes
    service := NewProductService(db).Build()
    mux := handlers.NewRouter(service, db).Build()  // ✅ Builder pattern for routes
    
    // ✅ Use protobuf structs for request
    reqData := &pb.CreateProductRequest{Name: "Test Product", Sku: "TEST-001"}
    body, _ := protojson.Marshal(reqData)
    req := httptest.NewRequest("POST", "/api/products", bytes.NewReader(body))
    rec := httptest.NewRecorder()
    
    mux.ServeHTTP(rec, req)  // ✅ Test through root mux
    
    if rec.Code != http.StatusCreated {
        t.Fatalf("Expected %d, got %d", http.StatusCreated, rec.Code)
    }
    
    var response pb.Product
    respBody, _ := io.ReadAll(rec.Body)
    protojson.Unmarshal(respBody, &response)
    
    // ✅ CORRECT: Build expected from fixtures, use cmp.Diff with protocmp
    expected := &pb.Product{
        Id:   response.Id,    // Random UUID (copy from response)
        Name: reqData.Name,   // From request fixture
        Sku:  reqData.Sku,    // From request fixture
    }
    
    // ✅ ALWAYS use cmp.Diff with protocmp.Transform() for protobuf comparison
    if diff := cmp.Diff(expected, &response, protocmp.Transform()); diff != "" {
        t.Errorf("Mismatch (-want +got):\n%s", diff)
    }
    
    // ❌ NEVER do individual field comparisons like this:
    // if response.Name != "Test Product" { t.Errorf(...) }
    // if !response.Valid { t.Errorf(...) }
}
```


## System Architecture

### X. Service Layer Architecture (Dependency Injection)

**Core Requirements**:
- Business logic MUST be Go interfaces (service layer)
- Services MUST NOT depend on HTTP types (only `context.Context` allowed)
- Handlers MUST be thin wrappers delegating to services
- Services and handlers MUST be in public packages (NOT `internal/`) for reusability
- External dependencies MUST be injected via builder pattern
- Services MUST use builder pattern: `NewService(db).WithLogger(log).Build()`
- `cmd/main.go` MUST ONLY call handlers or services (NEVER `internal/` packages directly)
- If `cmd/main.go` needs functionality from `internal/`, that code MUST be promoted to a public service

**Architecture**: `HTTP Handler (thin) → Service Interface (business logic) → Database`

**Rationale**: Separates transport from business logic, enables reuse across HTTP/gRPC/CLI/workers, allows external Go apps to import as library. Services in `internal/` cannot be imported externally (Go visibility rules). Main entry points calling internal packages creates tight coupling and breaks architectural boundaries - all application logic should flow through public services.

### Service Method Parameters: Protobuf Structs Only

**CRITICAL**: Service methods MUST use protobuf structs for ALL parameters and return types. NO primitives, NO maps.

```go
// ✅ CORRECT
type ProductService interface {
    Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error)
    Get(ctx context.Context, req *pb.GetProductRequest) (*pb.Product, error)
    List(ctx context.Context, req *pb.ListProductsRequest) (*pb.ListProductsResponse, error)
}

// ❌ WRONG
type ProductService interface {
    Create(ctx context.Context, name, sku string) (*pb.Product, error)  // NO primitives!
    Get(ctx context.Context, id string) (*pb.Product, error)            // NO primitives!
    List(ctx context.Context, limit, offset int) ([]*pb.Product, error) // NO multiple returns!
}
```

**Why Protobuf Only**:
- Built-in validation (no manual checks)
- Easy to add optional fields (no breaking changes)
- API documentation (protobuf comments)
- Cross-language support
- Type safety



**Package Structure**:

```go
// ✅ CORRECT: Services/handlers in public packages
github.com/theplant/myapp/
├── api/
│   ├── proto/pim/v1/  // PUBLIC - protobuf source files
│   │   └── product.proto
│   └── gen/pim/v1/    // PUBLIC - generated protobuf (services return these)
│       └── product.pb.go
├── services/          // PUBLIC - reusable by external apps
│   ├── product_service.go
│   └── errors.go
├── handlers/          // PUBLIC - reusable routing/HTTP logic
│   ├── product_handler.go
│   ├── error_codes.go
│   └── middleware.go
└── internal/          // INTERNAL - implementation only
    ├── models/        // ✅ OK (services return protobuf, not models)
    └── config/        // Not exposed

// ❌ WRONG: Services in internal (cannot import)
github.com/theplant/myapp/
└── internal/
    └── services/      // ❌ External apps cannot import!
```

**Why Models Stay Internal**:
Services return protobuf (public), use GORM models internally only. External apps never see models.

**When to Use `internal/`**:
- ✅ Models, middleware, config, database helpers
- ❌ Services, handlers, protobuf (must be public for reusability)


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
// Service interface
type ProductService interface {
    Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error)
    Get(ctx context.Context, req *pb.GetProductRequest) (*pb.Product, error)
}

type productService struct {
    db     *gorm.DB
    logger Logger  // Optional
    cache  Cache   // Optional
}

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

func (b *productServiceBuilder) Build() ProductService {
    return &productService{db: b.db, logger: b.logger, cache: b.cache}
}

// Service method
func (s *productService) Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    // Principle XIII: Validate and return sentinel errors
    if req.Name == "" {
        return nil, fmt.Errorf("product name: %w", ErrMissingRequired)
    }
    
    product := &Product{Name: req.Name, SKU: req.Sku}
    
    // Principle XII: Use context-aware database operations
    if err := s.db.WithContext(ctx).Create(product).Error; err != nil {
        // Principle XIII: Wrap errors with context
        if isDuplicateKeyError(err) {
            return nil, fmt.Errorf("create product SKU %s: %w", req.Sku, ErrDuplicateSKU)
        }
        return nil, fmt.Errorf("create product: %w", err)
    }
    
    // Principle V: Return protobuf type
    return &pb.Product{Id: product.ID, Name: product.Name, Sku: product.SKU}, nil
}
```

**Usage**:
```go
// Minimal (tests)
svc := NewProductService(db).Build()

// Production (with logging)
svc := NewProductService(db).WithLogger(log).WithCache(cache).Build()
```

**Optional: Service Middleware** (for advanced customization):
Services MAY expose middleware functions for external apps to add custom logic (validation, audit, rate-limiting). Builder pattern extends to support `WithCreateMiddleware(mw)` methods.

```go
// Usage: External apps add custom middleware
svc := services.NewProductService(db).
    WithCreateMiddleware(validateSKU).
    WithCreateMiddleware(auditLog).
    Build()
```


**HTTP Handler**:
```go
// Thin wrapper delegating to service
type ProductHandler struct {
    service ProductService
}

// Helper functions for protojson serialization (handlers/helpers.go)
func RespondWithProto(w http.ResponseWriter, status int, msg proto.Message) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    data, _ := protojson.Marshal(msg)  // ✅ Returns camelCase JSON
    w.Write(data)
}

func DecodeProtoJSON(r *http.Request, msg proto.Message) error {
    body, err := io.ReadAll(r.Body)
    if err != nil {
        return err
    }
    defer r.Body.Close()
    return protojson.Unmarshal(body, msg)  // ✅ Accepts camelCase JSON
}

func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req pb.CreateProductRequest
    if err := DecodeProtoJSON(r, &req); err != nil {  // ✅ Use protojson (accepts camelCase)
        RespondWithError(w, Errors.InvalidRequest)  // Principle XIII
        return
    }
    
    product, err := h.service.Create(r.Context(), &req)
    if err != nil {
        HandleServiceError(w, err)  // Principle XIII: Automatic error mapping
        return
    }
    
    RespondWithProto(w, http.StatusCreated, product)  // ✅ Use protojson (returns camelCase)
}

// Example handler using path parameter extraction
func (h *ProductHandler) Get(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    // Extract path parameter using r.PathValue() (Go 1.22+)
    id := r.PathValue("id")
    if id == "" {
        http.Error(w, "Missing product ID", http.StatusBadRequest)
        return
    }
    
    // Principle X: Service methods MUST use protobuf structs (NOT primitives)
    product, err := h.service.Get(ctx, &pb.GetProductRequest{Id: id})
    if err != nil {
        HandleServiceError(w, err)  // Principle XIII: Use error mapping
        return
    }
    
    RespondWithProto(w, http.StatusOK, product)  // ✅ Use protojson (returns camelCase)
}

```


**Routes Setup (Builder Pattern)**:

All dependencies optional. Middlewares must be added explicitly via variadic `WithMiddlewares()`.

```go
// handlers/routes.go
type Middleware func(http.Handler) http.Handler

type routerBuilder struct {
    mux              *http.ServeMux
    workflowService  services.WorkflowService
    activityService  services.ActivityService
    executionService services.ExecutionService
    triggerService   services.TriggerService
    db               *gorm.DB
    middlewares      []Middleware
}

func NewRouter(workflowService services.WorkflowService, db *gorm.DB) *routerBuilder {
    return &routerBuilder{workflowService: workflowService, db: db}
}

func (b *routerBuilder) WithMux(mux *http.ServeMux) *routerBuilder {
    b.mux = mux
    return b
}

func (b *routerBuilder) WithActivityService(svc services.ActivityService) *routerBuilder {
    b.activityService = svc
    return b
}

func (b *routerBuilder) WithExecutionService(svc services.ExecutionService) *routerBuilder {
    b.executionService = svc
    return b
}

func (b *routerBuilder) WithTriggerService(svc services.TriggerService) *routerBuilder {
    b.triggerService = svc
    return b
}

func (b *routerBuilder) WithMiddlewares(mws ...Middleware) *routerBuilder {
    b.middlewares = append(b.middlewares, mws...)
    return b
}

func (b *routerBuilder) Build() http.Handler {
    mux := b.mux
    if mux == nil {
        mux = http.NewServeMux()
    }
    
    // Register routes only for non-nil services
    if b.workflowService != nil {
        h := NewWorkflowHandler(b.workflowService)
        mux.HandleFunc("POST /api/v1/workflows", h.Create)
        mux.HandleFunc("GET /api/v1/workflows/{id}", h.Get)
        mux.HandleFunc("GET /api/v1/workflows", h.List)
        // ... more routes
    }
    if b.activityService != nil {
        h := NewActivityHandler(b.activityService)
        mux.HandleFunc("GET /api/v1/activities", h.List)
    }
    // ... other optional services
    
    // Apply middlewares in order
    var handler http.Handler = mux
    for _, mw := range b.middlewares {
        handler = mw(handler)
    }
    return handler
}

func DefaultMiddlewares() []Middleware {
    return []Middleware{
        recoveryMiddleware,
        LoggingMiddleware(),
        requestIDMiddleware,
        CORSMiddleware(DefaultCORSConfig()),
    }
}

// Usage: Tests (no middlewares needed)
mux := handlers.NewRouter(workflowService, db).Build()

// Usage: Production (explicit middlewares)
handler := handlers.NewRouter(workflowService, db).
    WithActivityService(activityService).
    WithExecutionService(executionService).
    WithMiddlewares(handlers.DefaultMiddlewares()...).
    Build()
```



## Distributed Tracing

### XI. Distributed Tracing (OpenTracing)

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
// HTTP Handler
func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    span := opentracing.StartSpan("POST /api/products")
    defer span.Finish()
    span.SetTag("http.method", r.Method)
    
    var req pb.CreateProductRequest
    if err := DecodeProtoJSON(r, &req); err != nil {  // ✅ Use protojson (accepts camelCase)
        span.SetTag("error", true)
        RespondWithError(w, Errors.InvalidRequest)
        return
    }
    
    product, err := h.service.Create(r.Context(), &req)
    if err != nil {
        span.SetTag("error", true)
        HandleServiceError(w, err)  // Principle XIII
        return
    }
    
    span.SetTag("http.status_code", http.StatusCreated)
    RespondWithProto(w, http.StatusCreated, product)  // ✅ Use protojson (returns camelCase)
}

// Service - child span
func (s *productService) Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    span, ctx := opentracing.StartSpanFromContext(ctx, "ProductService.Create")
    defer span.Finish()
    
    product := &Product{Name: req.Name, SKU: req.Sku}
    if err := s.db.WithContext(ctx).Create(product).Error; err != nil {  // Principle XII
        span.SetTag("error", true)
        return nil, err
    }
    
    return toProto(product), nil
}
```


## Context-Aware Operations

### XII. Context-Aware Operations

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
// Service - context as first parameter
type ProductService interface {
    Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error)
}

func (s *productService) Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    // Principle XII: Check cancellation before expensive operations
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }
    
    // Principle XIII: Validate and return sentinel errors
    if req.Name == "" {
        return nil, fmt.Errorf("product name: %w", ErrMissingRequired)
    }
    
    // Principle XII: Use context-aware database operations
    tx := s.db.WithContext(ctx).Begin()
    defer tx.Rollback()
    
    product := &Product{Name: req.Name, SKU: req.Sku}
    if err := tx.Create(product).Error; err != nil {
        // Principle XIII: Wrap errors with context
        return nil, fmt.Errorf("create product: %w", err)
    }
    
    if err := tx.Commit().Error; err != nil {
        return nil, fmt.Errorf("commit transaction: %w", err)
    }
    
    return toProto(product), nil
}

// HTTP Handler - extract context
func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()  // Principle XII: Extract context
    
    var req pb.CreateProductRequest
    if err := DecodeProtoJSON(r, &req); err != nil {  // ✅ Use protojson (accepts camelCase)
        RespondWithError(w, Errors.InvalidRequest)
        return
    }
    
    product, err := h.service.Create(ctx, &req)  // Propagate context
    if err != nil {
        HandleServiceError(w, err)  // Principle XIII: Handles context.Canceled automatically
        return
    }
    RespondWithProto(w, http.StatusCreated, product)  // ✅ Use protojson (returns camelCase)
}

// Long-running - check cancellation periodically
func (s *productService) BulkUpdate(ctx context.Context, updates []*pb.ProductUpdate) error {
    for i, update := range updates {
        if i%100 == 0 {
            select {
            case <-ctx.Done():
                return ctx.Err()
            default:
            }
        }
        // Process update...
    }
}
```



## Error Handling Strategy

### XIII. Comprehensive Error Handling

**Two-Layer Strategy**:
- **Service Layer**: Sentinel errors (package-level vars) with `fmt.Errorf("%w")` wrapping
- **HTTP Layer**: Singleton error code struct with automatic service error mapping
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
```proto
// api/proto/common/v1/error.proto - Define error response as protobuf
message ErrorResponse {
    string code = 1;     // e.g., "PRODUCT_NOT_FOUND"
    string message = 2;  // e.g., "Product not found"
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
    MissingRequired ErrorCode
    ProductNotFound ErrorCode
    DuplicateSKU    ErrorCode
    InternalError   ErrorCode
}{
    MissingRequired: ErrorCode{"MISSING_REQUIRED", "Required field missing", 400, services.ErrMissingRequired},
    ProductNotFound: ErrorCode{"PRODUCT_NOT_FOUND", "Product not found", 404, services.ErrProductNotFound},
    DuplicateSKU:    ErrorCode{"DUPLICATE_SKU", "SKU already exists", 409, services.ErrDuplicateSKU},
    InternalError:   ErrorCode{"INTERNAL_ERROR", "Internal server error", 500, nil},
}

// RespondWithError sends error response using protobuf ErrorResponse
func RespondWithError(w http.ResponseWriter, errCode ErrorCode) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(errCode.HTTPStatus)
    errResp := &pb.ErrorResponse{Code: errCode.Code, Message: errCode.Message}
    data, _ := protojson.Marshal(errResp)  // ✅ Use protojson for error responses too
    w.Write(data)
}
```

**Service - Wrap Errors**:
```go
func (s *productService) Get(ctx context.Context, req *pb.GetProductRequest) (*pb.Product, error) {
    var product Product
    if err := s.db.WithContext(ctx).First(&product, "id = ?", req.Id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("get product %s: %w", req.Id, ErrProductNotFound)
        }
        return nil, fmt.Errorf("query product %s: %w", req.Id, err)
    }
    return toProto(&product), nil
}
```

**Handler - Automatic Mapping**:
```go
func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req pb.CreateProductRequest
    if err := DecodeProtoJSON(r, &req); err != nil {  // ✅ Use protojson (accepts camelCase)
        RespondWithError(w, Errors.InvalidRequest)
        return
    }
    
    product, err := h.service.Create(r.Context(), &req)
    if err != nil {
        HandleServiceError(w, err)  // Automatic mapping!
        return
    }
    
    RespondWithProto(w, http.StatusCreated, product)  // ✅ Use protojson (returns camelCase)
}

// Automatically maps service errors to HTTP responses
func HandleServiceError(w http.ResponseWriter, err error) {
    // Check context errors first (Principle XII)
    if errors.Is(err, context.Canceled) {
        http.Error(w, "Request cancelled", 499)
        return
    }
    if errors.Is(err, context.DeadlineExceeded) {
        http.Error(w, "Request timeout", 504)
        return
    }
    
    // Check service error mapping
    for _, errCode := range AllErrors() {
        if errCode.ServiceErr != nil && errors.Is(err, errCode.ServiceErr) {
            RespondWithError(w, errCode)
            return
        }
    }
    RespondWithError(w, Errors.InternalError)
}
```

**Testing** (ALL errors MUST have test cases):
```go
func TestProductAPI_Create_DuplicateSKU(t *testing.T) {
    // Principle I: Integration test with real database
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    // Principle I: Create real database fixture
    createTestProduct(db, map[string]interface{}{"name": "Existing", "sku": "DUP-001"})
    
    // Principle X: Use builder pattern
    service := NewProductService(db).Build()
    mux := SetupRoutes(service)
    
    // Principle V: Use protobuf structs
    reqData := &pb.CreateProductRequest{Name: "New Product", Sku: "DUP-001"}
    body, _ := protojson.Marshal(reqData)  // ✅ Use protojson for protobuf (camelCase)
    req := httptest.NewRequest("POST", "/api/products", bytes.NewReader(body))
    rec := httptest.NewRecorder()
    
    // Principle IV: Test through root mux ServeHTTP
    mux.ServeHTTP(rec, req)
    
    // Verify HTTP status
    if rec.Code != http.StatusConflict {
        t.Errorf("Expected 409, got %d", rec.Code)
    }
    
    // Use error code definitions (NOT literal strings)
    var errResp pb.ErrorResponse
    respBody, _ := io.ReadAll(rec.Body)
    protojson.Unmarshal(respBody, &errResp)  // ✅ Use protojson for error response too
    
    if errResp.Code != Errors.DuplicateSKU.Code {
        t.Errorf("Expected %s, got %s", Errors.DuplicateSKU.Code, errResp.Code)
    }
}
```

**Error Assertions in Tests**:
- Tests MUST use `ErrorCode` definitions from `handlers/error_codes.go` (NOT literal strings)
- Tests MUST validate both `Code` and `Message` fields against definitions
- Example: `if errResp.Code != Errors.WorkflowNotFound.Code` (NOT `"workflow not found"`)

**Rationale**: Error code definitions provide single source of truth. Tests stay in sync when messages change.

**Key Benefits**:
- ✅ No switch statements (automatic mapping via ServiceErr field)
- ✅ Type-safe with `errors.Is()`
- ✅ Wrapping creates error breadcrumb trail
- ✅ ALL errors must be tested
- ✅ Error assertions use definitions (maintainable, refactoring-safe)


## Technology Stack

- **Language**: Go 1.25+ (recommend latest stable)
- **Database**: PostgreSQL 15+ (with JSONB support)
- **HTTP Framework**: Standard library `net/http` using `http.ServeMux`
- **Database Access**: GORM (gorm.io/gorm with gorm.io/driver/postgres)
- **Distributed Tracing**: OpenTracing (github.com/opentracing/opentracing-go)
- **Protocol Buffers**: protoc compiler, protoc-gen-go, protoc-gen-go-grpc
- **Protobuf JSON**: google.golang.org/protobuf/encoding/protojson (for camelCase JSON serialization)
- **Testing**: Standard library `testing` package with `httptest`
- **Test Comparison**: google/go-cmp with protocmp for protobuf message assertions
- **Test Database**: testcontainers-go with PostgreSQL module (automatic Docker container management)
- **Migration Tool**: GORM AutoMigrate (for development and testing)



## Code Review Requirements

All pull requests MUST be reviewed against these constitutional requirements:

**General Requirements**:
- Tests MUST be reviewed before implementation code (TDD workflow)
- Reviewers MUST verify GORM is used for database access (no raw SQL unless justified)
- ALL tests MUST pass before code review approval (no exceptions)
- Fixes MUST address root causes, not symptoms (Principle VIII)

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

**Version**: 1.10.0 | **Ratified**: 2025-11-20 | **Last Amended**: 2025-12-07

