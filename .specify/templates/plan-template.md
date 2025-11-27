# Implementation Plan: [FEATURE]

**Branch**: `[###-feature-name]` | **Date**: [DATE] | **Spec**: [link]
**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

[Extract from feature spec: primary requirement + technical approach from research]

## Technical Context

**Language/Version**: Go 1.25+ (recommend latest stable)  
**HTTP Framework**: Standard library `net/http` with `http.ServeMux`  
**Database**: PostgreSQL 15+ (with JSONB support)  
**Database Access**: GORM (`gorm.io/gorm` with `gorm.io/driver/postgres`)  
**Protocol Buffers**: `protoc` compiler, `protoc-gen-go`, `protoc-gen-go-grpc`  
**Validation**: `protoc-gen-validate` for protobuf field validation  
**Distributed Tracing**: OpenTracing (`github.com/opentracing/opentracing-go`)  
**Testing Framework**: Standard library `testing` package with `httptest`  
**Test Assertions**: `google/go-cmp` with `protocmp` for protobuf message comparison  
**Test Database**: `testcontainers-go` with PostgreSQL module (automatic Docker container management)  
**Migration Tool**: GORM AutoMigrate (for development and testing)

**Project Type**: [single/web/mobile - determines source structure]  
**Target Platform**: Linux server (containerized deployment)  
**Performance Goals**: [domain-specific, e.g., 1000 req/s, <100ms p95 latency or NEEDS CLARIFICATION]  
**Constraints**: [domain-specific, e.g., <200ms p95, <100MB memory or NEEDS CLARIFICATION]  
**Scale/Scope**: [domain-specific, e.g., 10k users, 1M requests/day or NEEDS CLARIFICATION]

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

[Gates determined based on constitution file]

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
├── proto/                  # Protobuf definitions (.proto files)
│   └── v1/
│       └── *.proto
└── gen/                    # Protobuf generated code (PUBLIC - must be importable)
    └── v1/
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
└── routes.go              # Shared routing configuration

internal/                   # INTERNAL - implementation details only
├── models/                # GORM models (internal - services return protobuf)
│   └── product.go
├── middleware/            # App-specific middleware (logging, CORS)
│   ├── logging.go
│   └── tracing.go
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
│   ├── proto/            # Protobuf definitions
│   └── gen/              # Generated code
├── services/              # PUBLIC - reusable
├── handlers/              # HTTP handlers (with *_test.go integration tests)
├── internal/              # Implementation details
├── testutil/              # Test helpers and fixtures
└── cmd/

service-b/
├── api/
│   ├── proto/            # Protobuf definitions
│   └── gen/              # Generated code
├── services/              # PUBLIC - reusable
├── handlers/              # HTTP handlers (with *_test.go integration tests)
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

**Critical Architecture Notes**:
- **Services MUST be in public packages** (`services/`, NOT `internal/services/`) to enable cross-application reuse per Constitution Principle XI
- **Handlers MUST be in public packages** (`handlers/`, NOT `internal/handlers/`) to enable cross-application reuse per Constitution Principle XI
- **Services return protobuf types** (`*pb.Product`), NOT internal GORM models, per Constitution Principle VI
- **Models CAN stay internal** (`internal/models/`) - only used internally for database mapping
- **Protobuf generated code MUST be public** (`api/gen/`) - external apps need these types to call services
- **Middleware SHOULD be internal** - application-specific implementation details
- **AutoMigrate() MUST be exported** in `services/migrations.go` for external apps to run schema migrations

## Testing Strategy

### Test-First Development (TDD)

Per Constitution Principle VII, this project follows strict TDD workflow:

1. **Design Phase**: Define API contracts in `.proto` files (request/response messages)
2. **Generate Code**: Run `protoc` to generate Go structs from protobuf definitions
3. **Write Tests**: Create table-driven integration tests using protobuf structs (NOT maps)
4. **Verify Failure**: Run tests to confirm they fail (red phase)
5. **Review Tests**: Review test design before implementation
6. **Implement**: Write minimal code to make tests pass (green phase)
7. **Run Tests**: Execute full test suite immediately after implementation (MANDATORY)
8. **Verify Success**: Confirm all tests pass (green phase)
9. **Refactor**: Improve code quality while keeping tests green
10. **Run Tests Again**: Execute tests after each refactoring change (MANDATORY)
11. **Complete Task**: Task is only complete when all tests pass

### Integration Testing Requirements

Per Constitution Principles I-VI:

- **Integration tests ONLY** - No mocking of database, HTTP clients, or external services
- **Real PostgreSQL** via `testcontainers-go` for automatic Docker container management
- **Table-driven design** - All tests use `[]struct` with `name` field
- **Comprehensive edge cases** (MANDATORY for every API endpoint):
  - **Input validation**: Empty strings, nil values, invalid formats, SQL injection attempts, XSS payloads
  - **Boundary conditions**: Zero values, negative numbers, maximum values, empty arrays, nil pointers
  - **Authentication/Authorization**: Missing tokens, expired tokens, invalid tokens, insufficient permissions
  - **Data state**: Non-existent resources (404), duplicate entries (conflict), concurrent modifications
  - **Database errors**: Constraint violations, foreign key failures, transaction conflicts
  - **HTTP specifics**: Wrong methods, missing headers, invalid content-types, malformed JSON
  - Edge cases MUST be documented in test case names
- **Real database fixtures** - Use GORM to insert data, truncate tables after tests with `defer`
- **ServeHTTP testing** - Test via `httptest.ResponseRecorder`, NOT bypassing HTTP layer
- **Protobuf structs** - NO `map[string]interface{}`, use generated types
- **Protobuf assertions** - Use `cmp.Diff()` with `protocmp.Transform()` for ALL message comparisons
- **Derive from fixtures** - Expected values from test fixtures (request data, DB fixtures), NOT response data (except UUIDs, timestamps, crypto-rand)
- **Continuous verification** - Run tests after EVERY code change (Principle VII)
- **Acceptance scenario coverage** - Each spec scenario (US#-AS#) has corresponding test case (Principle IX)
- **Test coverage analysis** - Use `go test -cover` to identify and close testing gaps (Principle X)

### Error Handling Strategy

Per Constitution Principle XIII:

**Service Layer (Internal)**:
- Define sentinel errors: `var ErrProductNotFound = errors.New("product not found")`
- Wrap errors with context: `fmt.Errorf("get product %s: %w", id, ErrProductNotFound)`
- Use `errors.Is()` and `errors.As()` for type-safe error checking

**HTTP Layer (Client-Facing)**:
- Define error code singleton in `handlers/error_codes.go`
- Map service errors to HTTP codes via `ErrorCode.ServiceErr` field
- Use `HandleServiceError()` for automatic error mapping (no switch statement)
- ALL sentinel errors MUST have test cases
- ALL HTTP error codes MUST have test cases

### Test Database Isolation

Per Constitution Development Workflow:

- **Testcontainers-go**: Automatic PostgreSQL container management (Docker required)
- **Database truncation**: `defer truncateTables(db, "products", "orders", "order_items")`
- **Truncate with CASCADE**: Handle foreign key dependencies
- **Each test gets clean state**: Truncation ensures isolation
- **Parallel test support**: `t.Parallel()` safe - each test gets own container

### Context-Aware Operations

Per Constitution Principle XIV:

- All service methods accept `context.Context` as first parameter
- HTTP handlers extract context from `r.Context()`
- Database operations use `db.WithContext(ctx)`
- External HTTP calls use `http.NewRequestWithContext(ctx, ...)`
- Long-running operations check `ctx.Done()` periodically
- Tests verify context cancellation behavior

### Distributed Tracing

Per Constitution Principle XII:

- Each HTTP endpoint creates OpenTracing span
- Service methods create child spans
- Database operations traced as single span per transaction (NOT per query)
- Error conditions tagged with `span.SetTag("error", true)`
- Development uses `opentracing.NoopTracer{}` (no external dependencies)
- Production configured via environment variables (Jaeger, Zipkin, etc.)

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
