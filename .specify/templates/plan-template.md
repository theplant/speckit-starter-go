# Implementation Plan: [FEATURE]

**Branch**: `[###-feature-name]` | **Date**: [DATE] | **Spec**: [link]
**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

[Extract from feature spec: primary requirement + technical approach from research]

## Technical Context

**Stack**: Go 1.25+, PostgreSQL 15+, GORM, Protobuf, OpenTracing, testcontainers-go  
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
- [ ] **V. Protobuf Data Structures (Generated from OpenAPI)**: API contracts in OpenAPI specs (single source of truth), `.proto` generated from OpenAPI via `openapi2proto` (MUST NOT hand-edit), tests use protobuf structs (NO `map[string]interface{}`), compare using `cmp.Diff()` with `protocmp.Transform()`, derive expected from TEST FIXTURES (NOT response except truly random: UUIDs, timestamps, crypto-rand tokens), stabilize field numbers via `x-proto-tag`
- [ ] **VI. Continuous Test Verification**: Run `go test -v ./...` after every change, pass before commit, fix failures immediately, run with `-race` for concurrency safety
- [ ] **VII. Root Cause Tracing**: Trace backward through call chain, distinguish symptoms from root causes, fix source NOT symptoms, NEVER remove/weaken tests, use debuggers/logging
- [ ] **VIII. Acceptance Scenario Coverage**: Every user scenario (US#-AS#) in spec.md has corresponding test, test case names reference scenarios, validate complete "Then" clause
- [ ] **IX. Test Coverage & Gap Analysis**: Run `go test -coverprofile=coverage.out ./...`, analyze with `go tool cover -func=coverage.out`, target >80%, remove dead code if unreachable

### System Architecture (X)
- [ ] **X. Service Layer Architecture**: Business logic in Go interfaces (service layer), services MUST NOT depend on HTTP types (only `context.Context` allowed), handlers thin wrappers
- [ ] **ALL Validation in Services**: Handlers MUST NOT validate input (including empty/nil checks). Handlers ONLY: decode request, extract path params, call service, write response
- [ ] **Package Structure**: Services/handlers in public packages (NOT `internal/`) for reusability, external dependencies injected via builder pattern
- [ ] **Builder Pattern**: Services use `NewService(db).WithLogger(log).Build()` (required params in constructor, optional via `With*()` methods). Routes use builder: `NewRouter(svc, db).WithMiddlewares(...).Build()`
- [ ] **Service Method Parameters**: MUST use protobuf structs for ALL parameters and return types (NO primitives, NO maps)
- [ ] **Main Entry Point**: `cmd/main.go` MUST ONLY call handlers or services (NEVER `internal/` packages directly). If needs `internal/` functionality, promote to public service
- [ ] **Data Flow**: HTTP → Handler (thin) → Service (protobuf) → GORM Model → Database

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
- [ ] **HTTP Layer**: Singleton error code struct in `handlers/error_codes.go` with `ServiceErr` field for automatic mapping via `HandleServiceError()`
- [ ] **Environment-Aware Details**: ErrorResponse includes `details` field with full error chain by default; hidden when `HIDE_ERROR_DETAILS=true`
- [ ] **RespondWithError Signature**: `RespondWithError(w, errCode, err)` - always pass original error for details
- [ ] **Startup Config**: Call `handlers.SetHideErrorDetails(true)` in `main.go` when env var set
- [ ] **Testing**: ALL errors (sentinel + HTTP codes) MUST have test cases
- [ ] **Error Assertions**: Tests use error code definitions (NOT literal strings)
- [ ] **Context Errors**: `HandleServiceError()` checks `context.Canceled` and `context.DeadlineExceeded` first

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
│   └── [SERVICE].json      # e.g., pim.json - API contract definitions
├── proto/                 # Protobuf definitions (GENERATED from OpenAPI - DO NOT HAND-EDIT)
│   └── [SERVICE]/v1/      # Service name with API versioning
│       └── *.proto        # Generated via openapi2proto
└── gen/                   # Protobuf generated code (PUBLIC - must be importable)
    └── [SERVICE]/v1/      # Service name with API versioning
        ├── *.pb.go
        └── *.pb.validate.go

services/                   # PUBLIC package - business logic (reusable by external apps)
├── product_service.go
├── errors.go              # Sentinel errors (ErrNotFound, ErrDuplicateSKU, etc.)
└── migrations.go          # AutoMigrate() function for external apps

handlers/                   # PUBLIC package - HTTP handlers (reusable)
├── product_handler.go
├── product_handler_test.go
├── error_codes.go         # HTTP error code singleton with ServiceErr mapping
├── helpers.go             # DecodeProtoJSON(), RespondWithProto(), RespondWithError()
├── middleware.go          # App-specific middleware
└── routes.go              # Shared routing configuration (builder pattern)

internal/                   # INTERNAL - implementation details only
├── models/                # GORM models (internal - services return protobuf)
│   └── product.go
└── config/                # Configuration loading

cmd/
└── api/                   # Main application entry point
    └── main.go

testutil/                   # Test helpers and fixtures
├── fixtures.go            # CreateTestXxx() functions with default values
└── db.go                  # setupTestDB() with testcontainers

# [REMOVE IF UNUSED] Option 2: Multiple Go services (microservices)
# Each service follows Option 1 structure
service-a/
├── api/
│   ├── openapi/          # OpenAPI specifications (SINGLE SOURCE OF TRUTH)
│   ├── proto/            # Protobuf definitions (GENERATED from OpenAPI)
│   │   └── [SERVICE]/v1/  # Service name with API versioning
│   └── gen/              # Generated code
│       └── [SERVICE]/v1/  # Service name with API versioning
├── services/              # PUBLIC - reusable
├── handlers/              # HTTP handlers (with *_test.go integration tests)
│   └── middleware.go     # App-specific middleware
├── internal/              # Implementation details
├── testutil/              # Test helpers and fixtures
└── cmd/

service-b/
├── api/
│   ├── openapi/          # OpenAPI specifications (SINGLE SOURCE OF TRUTH)
│   ├── proto/            # Protobuf definitions (GENERATED from OpenAPI)
│   │   └── [SERVICE]/v1/  # Service name with API versioning
│   └── gen/              # Generated code
│       └── [SERVICE]/v1/  # Service name with API versioning
├── services/              # PUBLIC - reusable
├── handlers/              # HTTP handlers (with *_test.go integration tests)
│   └── middleware.go     # App-specific middleware
├── internal/              # Implementation details
├── testutil/              # Test helpers and fixtures
└── cmd/

shared/                    # Shared libraries across services
├── tracing/
├── logging/
└── database/
```

**Structure Decision**: [Document the selected structure and reference the real
directories captured above]

**Architecture** (Constitution Principle X):
- Services/handlers: PUBLIC packages (return protobuf, reusable)
- Models: `internal/models/` (GORM only, never exposed)
- Protobuf: PUBLIC `api/gen/` (external apps need these)
- `AutoMigrate()`: Exported in `services/migrations.go`

## Testing Strategy

### Test-First Development (TDD)

TDD workflow (Constitution Development Workflow):

1. **Design**: Define API contract in OpenAPI → generate `.proto` with `openapi2proto` → generate Go code with `protoc`
2. **Red**: Write integration tests → verify FAIL
3. **Green**: Implement → run tests → verify PASS
4. **Refactor**: Improve code → run tests after each change
5. **Complete**: Done only when ALL tests pass

### OpenAPI → Protobuf Workflow

- **OpenAPI is single source of truth** for API contracts
- **Generate proto**: `openapi2proto -spec api/openapi/[domain].json -skip-rpcs -out api/proto/[domain]/v1/[domain].proto`
- **Stabilize field numbers**: Use `x-proto-tag` in OpenAPI (never reuse deleted field numbers)
- **Set protobuf options**: Use `x-global-options` in OpenAPI for `go_package`
- **NEVER hand-edit** generated `.proto` files (edit OpenAPI instead)

### Integration Testing Requirements

Constitution Testing Principles I-IX:

- **Integration tests ONLY** (NO mocking in implementation code), real PostgreSQL via testcontainers
- **Mocking Policy**: ONLY in test code (`*_test.go`) for external systems with justification, NEVER in production code
- **Test Setup**: Use public APIs and dependency injection (NOT direct `internal/` package imports). Exception: `testutil/` MAY import `internal/models` strictly for fixtures.
- **Table-driven** with `name` fields, execute using `t.Run(testCase.name, func(t *testing.T) {...})`
- **Edge cases MANDATORY**: Input validation (empty/nil, invalid formats, SQL injection, XSS), boundaries (zero/negative/max), auth (missing/expired/invalid tokens), data state (404s, conflicts), database (constraint violations, foreign key failures), HTTP (wrong methods, missing headers, invalid content-types, malformed JSON)
- **ServeHTTP testing** via root mux (NOT individual handlers), identical routing configuration from shared routes package
- **Protobuf** structs (NO `map[string]interface{}`), use `cmp.Diff()` with `protocmp.Transform()`
- **Derive from fixtures** (request data, database fixtures, config). Copy from response ONLY for truly random: UUIDs, timestamps, crypto-rand tokens. Read `testutil/fixtures.go` for defaults.
- **Run tests** after EVERY change: `go test -v ./...` and `go test -v -race ./...` (Principle VI)
- **Map scenarios** to tests (US#-AS# in test case names, Principle VIII)
- **Coverage >80%** for business logic (Principle IX), analyze gaps with `go tool cover -func=coverage.out`

### Error Handling Strategy

Constitution Principle XIII:

- **Service Layer**: Sentinel errors (package-level vars in `services/errors.go`, e.g., `var ErrProductNotFound = errors.New("product not found")`), wrap with `fmt.Errorf("context: %w", err)` for breadcrumb trail
- **HTTP Layer**: Singleton error code struct in `handlers/error_codes.go` with fields `Code`, `Message`, `HTTPStatus`, `ServiceErr` for automatic mapping
- **Automatic Mapping**: `HandleServiceError()` uses `errors.Is()` to check service errors and map to HTTP responses (NO switch statements)
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
