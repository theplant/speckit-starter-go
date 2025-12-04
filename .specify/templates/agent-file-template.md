# [PROJECT NAME] Development Guidelines

Auto-generated from all feature plans. Last updated: [DATE]

## Constitution Compliance

This project follows the Go Project Constitution, which defines core principles for:

### Testing Principles (I-IX)
- **I. Integration Testing (No Mocking)**: Real PostgreSQL database connections (NO mocking), test isolation with table truncation, fixtures via GORM, mocking ONLY in test code for external systems with justification
- **II. Table-Driven Design**: Test cases as slices of structs with descriptive `name` fields, execute using `t.Run(testCase.name, func(t *testing.T) {...})`
- **III. Comprehensive Edge Case Coverage**: Input validation (empty/nil, invalid formats, SQL injection, XSS), boundary conditions (zero/negative/max), auth (missing/expired/invalid tokens), data state (404s, conflicts, concurrent modifications), database (constraint violations, foreign key failures), HTTP (wrong methods, missing headers, invalid content-types, malformed JSON)
- **IV. ServeHTTP Endpoint Testing**: Call root mux ServeHTTP (NOT individual handlers), identical routing configuration, HTTP path patterns, `r.PathValue()` for parameters
- **V. Protobuf Data Structures**: API contracts in `.proto` files, tests use protobuf structs with `validate.rules`, compare using `cmp.Diff()` with `protocmp.Transform()`, derive expected values from TEST FIXTURES (NOT response except truly random fields: UUIDs, timestamps, crypto-rand tokens)
- **VI. Continuous Test Verification**: Run tests after EVERY code change, pass before commit, fix failures immediately, optimize if impacting velocity
- **VII. Root Cause Tracing**: Trace problems backward through call chain, distinguish symptoms from root causes, fix source NOT symptoms, NEVER remove/weaken tests
- **VIII. Acceptance Scenario Coverage**: Every user scenario (US#-AS#) in spec.md has corresponding automated test with scenario ID in test case name
- **IX. Test Coverage & Gap Analysis**: Run `go test -coverprofile=coverage.out`, analyze gaps with `go tool cover -func=coverage.out`, target >80% for business logic, remove dead code

### System Architecture (X)
- **X. Service Layer Architecture**: Business logic MUST be Go interfaces (service layer), services MUST NOT depend on HTTP types (only `context.Context` allowed), handlers MUST be thin wrappers delegating to services
- **Package Structure**: Services and handlers MUST be in public packages (NOT `internal/`) for reusability, external dependencies injected via builder pattern: `NewService(db).WithLogger(log).Build()`
- **Service Method Parameters**: MUST use protobuf structs for ALL parameters and return types (NO primitives, NO maps)
- **Main Entry Point**: `cmd/main.go` MUST ONLY call handlers or services (NEVER `internal/` packages directly). If `cmd/main.go` needs functionality from `internal/`, that code MUST be promoted to a public service

### Distributed Tracing (XI)
- **XI. Distributed Tracing (OpenTracing)**: HTTP endpoints MUST create OpenTracing spans with operation name (e.g., "POST /api/products")
- **Service Spans**: Service methods SHOULD create child spans (e.g., "ProductService.Create")
- **Database Operations**: ONE span per transaction (NOT per SQL query - too much overhead)
- **External Calls**: HTTP/gRPC calls MUST propagate trace context
- **Error Tagging**: Errors MUST set `span.SetTag("error", true)`
- **Span Tags**: MUST include `http.method`, `http.url`, `http.status_code`
- **Setup**: Development/Tests use `opentracing.NoopTracer{}`, Production configured from environment variables (Jaeger, Zipkin, Datadog, etc.)

### Context-Aware Operations (XII)
- **XII. Context-Aware Operations**: Service methods MUST accept `context.Context` as first parameter
- **HTTP Handlers**: MUST use `r.Context()`
- **Database Operations**: MUST use `db.WithContext(ctx)`
- **External HTTP Calls**: MUST use `http.NewRequestWithContext(ctx, ...)`
- **Long-Running Operations**: MUST check context cancellation periodically using `select { case <-ctx.Done(): return ctx.Err() }`
- **Tests**: MUST verify context cancellation behavior
- **Rationale**: Enables timeout handling, graceful cancellation, trace propagation, prevents resource leaks

### Error Handling Strategy (XIII)
- **XIII. Comprehensive Error Handling - Two-Layer Strategy**:
  - **Service Layer**: Sentinel errors (package-level vars like `ErrProductNotFound`) with `fmt.Errorf("%w")` wrapping for error breadcrumb trail
  - **HTTP Layer**: Singleton error code struct with automatic service error mapping via `HandleServiceError()` function (NO switch statements)
  - **Testing**: ALL errors (sentinel + HTTP codes) MUST have test cases
- **Error Code Struct**: `type ErrorCode struct { Code string; Message string; HTTPStatus int; ServiceErr error }`
- **Automatic Mapping**: HTTP layer uses `errors.Is()` to check service errors and automatically map to HTTP responses
- **Error Assertions**: Tests MUST use `ErrorCode` definitions from `handlers/error_codes.go` (NOT literal strings)
- **Context Errors**: `HandleServiceError()` checks `context.Canceled` and `context.DeadlineExceeded` first
- **Rationale**: Type-safe, maintainable, wrapping creates breadcrumb trail, single source of truth for error messages

## Active Technologies

[EXTRACTED FROM ALL PLAN.MD FILES]

## Project Structure

```text
[ACTUAL STRUCTURE FROM PLANS]
```

### Key Architecture Principles

- **Services/handlers**: PUBLIC packages (return protobuf, reusable)
- **Models**: `internal/models/` (GORM only, never exposed)
- **Protobuf**: PUBLIC `api/gen/` (external apps need these)
- **AutoMigrate()**: Exported in `services/migrations.go`

## Commands

[ONLY COMMANDS FOR ACTIVE TECHNOLOGIES]

## Testing Workflow

### Test-First Development (TDD)

1. **Design**: Define API in `.proto` files → generate code
2. **Red**: Write integration tests → verify FAIL
3. **Green**: Implement → run tests → verify PASS
4. **Refactor**: Improve code → run tests after each change
5. **Complete**: Done only when ALL tests pass

### Test Requirements

- **Integration tests ONLY** (NO mocking in implementation code), real PostgreSQL via testcontainers
- **Mocking Policy**: Mocking ONLY permitted in test code (`*_test.go`) for external systems (third-party APIs, message queues) with written justification. Mock implementations NEVER in production code files.
- **Test Setup**: MUST use public APIs and dependency injection (NOT direct `internal/` package imports)
- **Table-driven** with `name` fields, execute using `t.Run(testCase.name, func(t *testing.T) {...})`
- **Edge cases MANDATORY**: Input validation (empty/nil, invalid formats, SQL injection, XSS), boundary conditions (zero/negative/max), auth (missing/expired/invalid tokens), data state (404s, conflicts), database (constraint violations, foreign key failures), HTTP (wrong methods, missing headers, invalid content-types, malformed JSON)
- **ServeHTTP testing** via root mux (NOT individual handlers), identical routing configuration from shared routes package
- **Protobuf** structs with `validate.rules`, use `cmp.Diff()` with `protocmp.Transform()` (NO `==`, `reflect.DeepEqual`, or individual field checks)
- **Derive from fixtures** (request data, database fixtures, config). Copy from response ONLY for truly random fields: UUIDs, timestamps, crypto-rand tokens. Read `testutil/fixtures.go` to find `CreateTestXxx()` defaults before writing assertions.
- **Run tests** after EVERY change with `go test -v ./...` and `go test -v -race ./...` for concurrency safety
- **Map scenarios** to tests (US#-AS# in test case names)
- **Coverage >80%** for business logic, run `go test -coverprofile=coverage.out ./...` and analyze gaps

## Code Style

[LANGUAGE-SPECIFIC, ONLY FOR LANGUAGES IN USE]

### Go Style Guidelines

**Service Layer**:
- Services MUST be Go interfaces in public packages (NOT `internal/`)
- Services MUST use builder pattern: `NewService(db).WithLogger(log).Build()` (required params in constructor, optional params via `With*()` methods)
- Service methods MUST use protobuf structs for ALL parameters and return types (NO primitives, NO maps)
- Service methods MUST accept `context.Context` as first parameter
- Service methods MUST use `db.WithContext(ctx)` for database operations
- Services MUST return protobuf types (NEVER internal GORM models)

**Error Handling**:
- Service layer: Define sentinel errors in `services/errors.go` (e.g., `var ErrProductNotFound = errors.New("product not found")`)
- Service layer: Wrap errors with `fmt.Errorf("context: %w", err)` to create breadcrumb trail
- HTTP layer: Define error codes in `handlers/error_codes.go` with `ServiceErr` field for automatic mapping
- HTTP layer: Use `HandleServiceError()` for automatic service error → HTTP response mapping
- Tests: MUST use error code definitions (NOT literal strings)

**Testing**:
- Tests MUST use table-driven design with descriptive `name` fields
- Tests MUST call root mux ServeHTTP (NOT individual handlers)
- Tests MUST use `cmp.Diff()` with `protocmp.Transform()` for protobuf assertions
- Tests MUST derive expected values from fixtures (NOT response, except UUIDs/timestamps/crypto-rand tokens)
- Tests MUST map to acceptance scenarios (US#-AS# in test case names)
- Tests MUST run after EVERY code change

**Handlers**:
- Handlers MUST be thin wrappers delegating to services
- Handlers MUST extract `r.Context()` and pass to services
- Handlers MUST use `r.PathValue()` for path parameters (NOT string manipulation)
- Handlers MUST use `HandleServiceError()` for automatic error mapping

**Package Structure**:
- Services/handlers: PUBLIC packages (external apps can import)
- Models: `internal/models/` (GORM only, never exposed)
- Protobuf: PUBLIC `api/gen/` (external apps need these)
- Export `AutoMigrate()` in `services/migrations.go` for external apps

## Recent Changes

[LAST 3 FEATURES AND WHAT THEY ADDED]

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
