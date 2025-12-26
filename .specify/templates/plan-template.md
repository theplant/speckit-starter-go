# Implementation Plan: [FEATURE]

**Branch**: `[###-feature-name]` | **Date**: [DATE] | **Spec**: [link]
**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

[Extract from feature spec: primary requirement + technical approach from research]

## Technical Context

**Stack**: Go 1.22+, PostgreSQL 15+, GORM, ogen (OpenAPI), OpenTracing, testcontainers-go  
**Project Type**: [single/web/mobile]  
**Target**: Linux server (containerized)  
**Performance**: [e.g., 1000 req/s, <100ms p95 or NEEDS CLARIFICATION]  
**Scale**: [e.g., 10k users, 1M req/day or NEEDS CLARIFICATION]

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Testing Principles (I-IX)
- [ ] **I. Integration Testing (No Mocking)**: Real PostgreSQL via testcontainers, test isolation with table truncation, fixtures via GORM, NO mocks in implementation code, mocking ONLY in `*_test.go` for external systems with justification. Test setup uses public APIs and dependency injection (NOT direct `internal/` package imports). Exception: `testutil/` MAY import `internal/models` strictly for fixtures.
- [ ] **II. Table-Driven Design**: Test cases as slices of structs with descriptive `name` fields, execute using `t.Run(testCase.name, func(t *testing.T) {...})`
- [ ] **III. Comprehensive Edge Case Coverage**: Input validation (empty/nil, invalid formats, SQL injection, XSS), boundaries (zero/negative/max), auth (missing/expired/invalid tokens), data state (404s, conflicts), database (constraint violations, foreign key failures), HTTP (wrong methods, missing headers, invalid content-types, malformed JSON)
- [ ] **IV. ServeHTTP Endpoint Testing**: Call root mux ServeHTTP (NOT individual handlers) using `httptest.ResponseRecorder`, identical routing configuration, HTTP path patterns, `r.PathValue()` for parameters
- [ ] **V. API Data Structures (Schema-First Design)**: OpenAPI with ogen
  - API contracts in OpenAPI specs (`api/openapi/<domain>.yaml`)
  - Go code generated via `ogen` (`go run github.com/ogen-go/ogen/cmd/ogen@latest`)
  - **Services implement generated `Handler` interface in `services/` package (NEVER in handlers/)**
  - Tests use generated structs (NO `map[string]interface{}`), compare using `cmp.Diff()`
  - Derive expected from TEST FIXTURES (NOT response except truly random: UUIDs, timestamps, crypto-rand tokens)
- [ ] **VI. Continuous Test Verification**: Run `go test -v ./...` after every change, pass before commit, fix failures immediately, run with `-race` for concurrency safety
- [ ] **VII. Root Cause Tracing**: Trace backward through call chain, distinguish symptoms from root causes, fix source NOT symptoms, NEVER remove/weaken tests, use debuggers/logging
- [ ] **VIII. Acceptance Scenario Coverage**: Every user scenario (US#-AS#) in spec.md has corresponding test, test case names reference scenarios, validate complete "Then" clause
- [ ] **IX. Test Coverage & Gap Analysis**: Run `go test -coverprofile=coverage.out ./...`, analyze with `go tool cover -func=coverage.out`, target >80%, remove dead code if unreachable

### System Architecture (X)
- [ ] **X. Service Layer Architecture (OgenHandler Delegation Pattern)**:
  - **`services/ogen_handler.go`**: Implements ogen-generated `Handler` interface and delegates to domain services
  - **Domain Services**: Have interfaces (e.g., `ProductService`) with implementations containing business logic
  - **`handlers/routes.go`**: Provides router builder that creates ogen server with ErrorHandler
- [ ] **OgenHandler**: Handles nil-service checks centrally
- [ ] **Services**: Reusable Go packages with OpenAPI-defined interfaces, MUST NOT depend on HTTP types (only `context.Context` allowed)
- [ ] **ALL Validation in Services**: ogen provides schema validation, services add business validation
- [ ] **Package Structure**: Services in public packages (NOT `internal/`) for reusability
- [ ] **Builder Pattern**: Services use `NewService(db).WithLogger(log).Build()` (required params in constructor, optional via `With*()` methods)
- [ ] **Service Method Parameters**: MUST use ogen-generated structs for ALL parameters and return types (NO primitives, NO maps)
- [ ] **Main Entry Point**: `cmd/main.go` MUST use `handlers.NewRouter().Build()` (NOT `api.NewServer()` directly). If needs `internal/` functionality, promote to public service
- [ ] **Data Flow**: HTTP (ogen server with ErrorHandler) → OgenHandler (delegates) → Domain Service → GORM Model → Database
- [ ] **handlers/ Package**: Provides router builder with ErrorHandler configuration, error code definitions, error mapping

### Distributed Tracing (XI)
- [ ] **XI. Distributed Tracing (OpenTracing)**: HTTP endpoints MUST create OpenTracing spans with operation name (e.g., "POST /api/products")
- [ ] **Service Spans**: Service methods SHOULD create child spans (e.g., "ProductService.Create")
- [ ] **Database Operations**: ONE span per transaction (NOT per SQL query - too much overhead)
- [ ] **External Calls**: HTTP/gRPC MUST propagate trace context
- [ ] **Error Tagging**: Errors MUST set `span.SetTag("error", true)`
- [ ] **Span Tags**: MUST include `http.method`, `http.url`, `http.status_code`
- [ ] **Setup**: Development/Tests use `opentracing.NoopTracer{}`, Production configured from env vars (Jaeger, Zipkin, Datadog)

### Context-Aware Operations (XII)
- [ ] **XII. Context-Aware Operations**: Service methods MUST accept `context.Context` as first parameter
- [ ] **HTTP Handlers**: MUST use `r.Context()`
- [ ] **Database Operations**: MUST use `db.WithContext(ctx)`
- [ ] **External HTTP Calls**: MUST use `http.NewRequestWithContext(ctx, ...)`
- [ ] **Long-Running Operations**: MUST check context cancellation periodically (`select { case <-ctx.Done(): return ctx.Err() }`)
- [ ] **Tests**: MUST verify context cancellation behavior

### Error Handling Strategy (XIII)
- [ ] **XIII. Comprehensive Error Handling**: Two-layer strategy (service + HTTP) with environment-aware details
- [ ] **Service Layer**: Sentinel errors (package-level vars in `services/errors.go`) with `fmt.Errorf("%w")` wrapping for breadcrumb trail
- [ ] **HTTP Layer**: ErrorResponse MUST be defined in OpenAPI schema (generated by ogen), error codes in `handlers/error_codes.go`
- [ ] **ErrorHandler**: `handlers/error_handler.go` implements ogen ErrorHandler interface, maps service errors to HTTP responses
- [ ] **Server Setup**: `handlers/server.go` provides `NewServer()` that configures `api.WithErrorHandler(OgenErrorHandler)`
- [ ] **Environment-Aware Details**: ErrorResponse includes `details` field with full error chain by default; hidden when `HIDE_ERROR_DETAILS=true`
- [ ] **Startup Config**: Call `handlers.SetHideErrorDetails(true)` in `main.go` when env var set
- [ ] **Testing**: ALL errors (sentinel + HTTP codes) MUST have test cases
- [ ] **Error Assertions**: Tests use error code definitions (NOT literal strings)
- [ ] **Context Errors**: ErrorHandler checks `context.Canceled` and `context.DeadlineExceeded` first

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)
<!--
  ACTION REQUIRED: Replace the placeholder tree below with the concrete layout
  for this feature. Delete unused options and expand the chosen structure with
  real paths (e.g., cmd/api, cmd/worker). The delivered plan must not include
  Option labels.
-->

```text
# [REMOVE IF UNUSED] Option 1: Single Go API (DEFAULT)
# Services MUST be in public packages (not internal/) for cross-app reusability
api/
├── openapi/               # OpenAPI specifications (SINGLE SOURCE OF TRUTH)
│   └── [SERVICE].yaml      # e.g., pim.yaml - API contract definitions (YAML preferred)
└── gen/                   # ogen generated code (PUBLIC - must be importable)
    └── [SERVICE]/         # Service name
        ├── gen.go             # go:generate directive
        ├── oas_handlers_gen.go
        ├── oas_server_gen.go
        └── oas_schemas_gen.go

services/                   # PUBLIC package - OgenHandler + domain services
├── ogen_handler.go        # Implements api.Handler, delegates to domain services
├── product_service.go     # Domain service interface + implementation (business logic)
├── errors.go              # Sentinel errors (ErrNotFound, ErrDuplicateSKU, etc.)
└── migrations.go          # AutoMigrate() function for external apps

handlers/                   # PUBLIC package - router setup with ErrorHandler
├── routes.go              # NewRouter() builder that creates ogen server with ErrorHandler
├── error_handler.go       # ogen ErrorHandler implementation (OgenErrorHandler)
└── error_codes.go         # Error code definitions mapping to services.Err*

tests/                      # SEPARATE package for integration tests
├── testutil_test.go       # Test helpers (setupTestDB, truncateTables, fixtures)
└── product_test.go        # Integration tests for product API

internal/                   # INTERNAL - implementation details only
├── models/                # GORM models (internal - services return generated types)
│   └── product.go
└── config/                # Configuration loading

cmd/
└── api/                   # Main application entry point
    └── main.go

# [REMOVE IF UNUSED] Option 2: Multiple Go services (microservices)
# Each service follows Option 1 structure
service-a/
├── api/
│   ├── openapi/          # OpenAPI specifications (SINGLE SOURCE OF TRUTH)
│   └── gen/              # ogen generated code
│       └── [SERVICE]/    # Service name
├── services/              # OgenHandler + domain services
├── handlers/              # Router builder with ErrorHandler
├── tests/                 # SEPARATE package for integration tests
├── internal/              # Implementation details
└── cmd/

service-b/
├── api/
│   ├── openapi/          # OpenAPI specifications (SINGLE SOURCE OF TRUTH)
│   └── gen/              # ogen generated code
│       └── [SERVICE]/    # Service name
├── services/              # OgenHandler + domain services
├── handlers/              # Router builder with ErrorHandler
├── tests/                 # SEPARATE package for integration tests
├── internal/              # Implementation details
└── cmd/

shared/                    # Shared libraries across services
├── tracing/
├── logging/
└── database/
```

**Structure Decision**: [Document the selected structure and reference the real
directories captured above]

**Architecture** (Constitution Principle X - OgenHandler Delegation Pattern):
- **OgenHandler**: `services/ogen_handler.go` implements ogen `Handler` interface, delegates to domain services
- **Domain Services**: `services/` package contains interfaces (e.g., `ProductService`) with business logic implementations
- **Router Builder**: `handlers/routes.go` creates ogen server with ErrorHandler
- Models: `internal/models/` (GORM only, never exposed)
- Generated types: PUBLIC `api/gen/` (external apps need these)
- `AutoMigrate()`: Exported in `services/migrations.go`
- ErrorResponse: MUST be defined in OpenAPI schema (generated by ogen)
- External apps can import domain services directly or use OgenHandler for full API compatibility

## Testing Strategy

### Test-First Development (TDD)

TDD workflow (Constitution Development Workflow):

1. **Design**: Define API contract (OpenAPI) → generate Go code with ogen
2. **Red**: Write integration tests → verify FAIL
3. **Green**: Implement → run tests → verify PASS
4. **Refactor**: Improve code → run tests after each change
5. **Complete**: Done only when ALL tests pass

### TDD Cycle (Same Pattern)

- [ ] T080 [US2] Define API contract (OpenAPI) → Generate Go code with ogen
- [ ] T081 [US2] Write tests (US2-AS1, US2-AS2 + edge cases) → Verify FAIL 
- [ ] T082 [US2] Implement (model, errors, domain service, ogen_handler, routes) → RUN TESTS → Verify PASS 
- [ ] T083 [US2] Refactor → Run tests after each change 
- [ ] T084 [US2] Coverage analysis → Verify >80%

### API Contract Workflow (OpenAPI with ogen)

- Define API contract in `api/openapi/<domain>.yaml` (YAML preferred)
- Add go:generate directive in `api/gen/<domain>/gen.go`:
  ```go
  package <domain>
  
  //go:generate go run github.com/ogen-go/ogen/cmd/ogen@latest --target . --clean ../../openapi/<domain>.yaml
  ```
- Generate Go code: `go generate ./...`
- **Create `services/ogen_handler.go`** that implements ogen `Handler` interface and delegates to domain services
- **Create domain services** with interfaces (e.g., `ProductService`) containing business logic
- **Create `handlers/routes.go`** with router builder that creates ogen server with ErrorHandler
- Use generated types in services and tests
- Services are reusable Go packages with OpenAPI-defined interfaces
- External apps can import domain services or use OgenHandler for full API compatibility

### Integration Testing Requirements

Constitution Testing Principles I-IX:

- **Integration tests ONLY** (NO mocking in implementation code), real PostgreSQL via testcontainers
- **Mocking Policy**: ONLY in test code (`*_test.go`) for external systems with justification, NEVER in production code
- **Test Setup**: Use public APIs and dependency injection (NOT direct `internal/` package imports). Exception: `testutil/` MAY import `internal/models` strictly for fixtures.
- **Table-driven** with `name` fields, execute using `t.Run(testCase.name, func(t *testing.T) {...})`
- **Edge cases MANDATORY**: Input validation (empty/nil, invalid formats, SQL injection, XSS), boundaries (zero/negative/max), auth (missing/expired/invalid tokens), data state (404s, conflicts), database (constraint violations, foreign key failures), HTTP (wrong methods, missing headers, invalid content-types, malformed JSON)
- **ServeHTTP testing** via root mux (NOT individual handlers), identical routing configuration from shared routes package
- **Generated** structs (NO `map[string]interface{}`), use `cmp.Diff()` for comparisons
- **Derive from fixtures** (request data, database fixtures, config). Copy from response ONLY for truly random: UUIDs, timestamps, crypto-rand tokens. Read `testutil/fixtures.go` for defaults.
- **Run tests** after EVERY change: `go test -v ./...` and `go test -v -race ./...` (Principle VI)
- **Map scenarios** to tests (US#-AS# in test case names, Principle VIII)
- **Coverage >80%** for business logic (Principle IX), analyze gaps with `go tool cover -func=coverage.out`

### Error Handling Strategy

Constitution Principle XIII:

- **Service Layer**: Sentinel errors (package-level vars in `services/errors.go`, e.g., `var ErrProductNotFound = errors.New("product not found")`), wrap with `fmt.Errorf("context: %w", err)` for breadcrumb trail
- **HTTP Layer**: ErrorResponse MUST be defined in OpenAPI schema (generated by ogen), error codes in `handlers/error_codes.go`
- **ErrorHandler**: `handlers/error_handler.go` implements ogen ErrorHandler, maps service errors to HTTP responses via `errors.Is()`
- **Router Setup**: `handlers/routes.go` provides `NewRouter()` builder that configures `api.WithErrorHandler(OgenErrorHandler)`
- **Context Errors**: Check `context.Canceled` (499) and `context.DeadlineExceeded` (504) first
- **Testing**: ALL errors (sentinel + HTTP codes) MUST have test cases
- **Error Assertions**: Tests MUST use error code definitions from `handlers/error_codes.go` (NOT literal strings)

### Test Database Isolation

- **testcontainers-go** with PostgreSQL (Docker required)
- **Truncation**: `defer truncateTables(db, "tables...")` with CASCADE
- **Truncate in reverse dependency order** (children before parents)
- **Centralize truncation logic** in helper function
- **Parallel**: `t.Parallel()` safe

### Context-Aware Operations

Constitution Principle XII:
- Service methods MUST accept `context.Context` as first parameter
- HTTP handlers MUST use `r.Context()` and pass to services
- Database operations MUST use `db.WithContext(ctx)`
- External HTTP calls MUST use `http.NewRequestWithContext(ctx, ...)`
- Long-running operations MUST check context cancellation periodically
- Tests MUST verify context cancellation behavior
- **Rationale**: Enables timeout handling, graceful cancellation, trace propagation, prevents resource leaks

### Distributed Tracing

Constitution Principle XI:
- HTTP endpoints MUST create OpenTracing spans with operation name (e.g., "POST /api/products")
- Service methods SHOULD create child spans (e.g., "ProductService.Create")
- Database operations: ONE span per transaction (NOT per SQL query - too much overhead)
- External calls (HTTP, gRPC) MUST propagate trace context
- Errors MUST set `span.SetTag("error", true)`
- Spans MUST include tags: `http.method`, `http.url`, `http.status_code`
- Development/Tests: Use `opentracing.NoopTracer{}`
- Production: Configure from environment variables (Jaeger, Zipkin, Datadog, etc.)
- **Rationale**: Provides observability for debugging latency, understanding request flows, identifying bottlenecks

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
