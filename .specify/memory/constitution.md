<!--
SYNC IMPACT REPORT
Version: 1.6.3 → 1.6.4
Modified Principles: None
Changes:
- Updated error-handling.md "Error Testing Requirements" to follow testing-guidelines.md patterns
- Error test examples now demonstrate constitutional testing principles (I-VI, XI)
- Examples show: mux.ServeHTTP (not handler methods), builder pattern, protobuf structs, real fixtures
- Added complete integration test example for error scenarios
- Added table-driven test example for multiple validation errors
- All error testing now consistent with constitutional requirements
Added Sections:
- error-handling.md: Comprehensive error testing examples following testing-guidelines.md
Removed Sections: None
Templates Requiring Updates: None
Follow-up TODOs: None
-->
# Go Project Constitution

This constitution defines the core principles and governance for Go microservice development. Detailed implementation guidelines, examples, and rationale are provided in separate appendix documents.

## Core Principles

### Testing Principles (I-X)

**Summary**: All code MUST be validated through comprehensive integration tests that use real dependencies (no mocking), follow table-driven patterns, cover edge cases, and map directly to acceptance scenarios. Tests MUST be run continuously after every code change.

**Key Requirements**:
- Integration tests with real PostgreSQL (testcontainers)
- Table-driven test design
- Comprehensive edge case coverage
- Real database fixtures
- ServeHTTP endpoint testing through full HTTP stack
- Protobuf data structures (no maps)
- Continuous test verification after every code change
- Root cause tracing for debugging
- Acceptance scenario coverage (spec-to-test mapping)
- Test coverage gap analysis

**Detailed Guidelines**: See [Testing Guidelines Appendix](./appendices/testing-guidelines.md)

### XI. Service Layer Architecture (Dependency Injection)

**Summary**: Business logic MUST be separated from HTTP transport using service interfaces. Services and handlers MUST be in public packages (not `internal/`) to enable cross-application reuse. Services MUST accept protobuf structs as parameters and return protobuf types. Dependencies are injected via builder pattern.

**Key Requirements**:
- Services in public packages for Go library reusability
- **Handlers in public packages for cross-application reuse**
- Services MUST NOT depend on HTTP types
- **Service method parameters MUST be protobuf structs** (no primitive types, no maps)
- **Service method return types MUST be protobuf structs** (models stay internal)
- HTTP handlers are thin wrappers
- Dependency injection via builder pattern
- Middleware pattern for extensibility

**Detailed Guidelines**: See [Service Architecture Appendix](./appendices/service-architecture.md)

### XII. Distributed Tracing (OpenTracing)

**Summary**: All API endpoints MUST be instrumented with distributed tracing at appropriate granularity (HTTP endpoints, service operations, database transactions - but NOT individual SQL queries).

**Key Requirements**:
- OpenTracing spans for HTTP endpoints and service methods
- Trace context propagation
- Error tagging in spans
- Tests verify tracing instrumentation
- Development uses NoopTracer

**Detailed Guidelines**: See [Observability Appendix](./appendices/observability.md#xii-distributed-tracing-opentracing)

### XIII. Comprehensive Error Handling

**Summary**: Error handling MUST use a two-layer strategy: sentinel errors for internal flow and singleton HTTP error codes for client responses. All errors MUST be wrapped to preserve error chains.

**Key Requirements**:
- Sentinel errors for domain-specific error types
- HTTP error code singleton with ServiceErr mapping
- Error wrapping with `fmt.Errorf("%w", err)`
- Type-safe checking with `errors.Is()` and `errors.As()`
- ALL defined errors MUST have test cases

**Detailed Guidelines**: See [Error Handling Appendix](./appendices/error-handling.md)

### XIV. Context-Aware Operations

**Summary**: All I/O and long-running operations MUST accept and respect `context.Context` for timeout handling, cancellation, and trace propagation.

**Key Requirements**:
- Service methods accept context as first parameter
- Database operations use `db.WithContext(ctx)`
- External calls propagate context
- Tests verify context cancellation behavior

**Detailed Guidelines**: See [Observability Appendix](./appendices/observability.md#xiv-context-aware-operations)

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



## Code Review Requirements

All pull requests MUST be reviewed against these constitutional requirements:

**General Requirements**:
- Tests MUST be reviewed before implementation code (TDD workflow)
- Reviewers MUST verify GORM is used for database access (no raw SQL unless justified)
- ALL tests MUST pass before code review approval (no exceptions)
- Fixes MUST address root causes, not symptoms (Principle VIII)

**Detailed Review Checklists**: Each appendix contains specific code review requirements for its domain:
- [Testing Principles Review Checklist](./appendices/testing-guidelines.md#code-review-requirements)
- [Service Architecture Review Checklist](./appendices/service-architecture.md#code-review-requirements)
- [Observability Review Checklist](./appendices/observability.md#code-review-requirements)
- [Error Handling Review Checklist](./appendices/error-handling.md#code-review-requirements)

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

**Version**: 1.6.4 | **Ratified**: 2025-11-20 | **Last Amended**: 2025-11-27

---

## Appendices

The following appendices provide detailed implementation guidelines, code examples, rationale, and code review checklists:

1. **[Testing Guidelines](./appendices/testing-guidelines.md)** - Principles I-X covering integration testing, table-driven tests, edge cases, fixtures, ServeHTTP testing, protobuf structures, continuous verification, debugging, acceptance scenarios, and coverage analysis

2. **[Service Architecture](./appendices/service-architecture.md)** - Principle XI covering dependency injection, service interfaces, package structure, middleware patterns, and cross-application reusability

3. **[Observability](./appendices/observability.md)** - Principles XII & XIV covering distributed tracing (OpenTracing) and context-aware operations for timeouts, cancellation, and trace propagation

4. **[Error Handling](./appendices/error-handling.md)** - Principle XIII covering two-layer error strategy with sentinel errors and HTTP error code singletons

