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

**Architecture** (Constitution Principle X):
- Services/handlers: PUBLIC packages (return protobuf, reusable)
- Models: `internal/models/` (GORM only, never exposed)
- Protobuf: PUBLIC `api/gen/` (external apps need these)
- `AutoMigrate()`: Exported in `services/migrations.go`

## Testing Strategy

### Test-First Development (TDD)

TDD workflow (Constitution Development Workflow):

1. **Design**: Define API in `.proto` files → generate code
2. **Red**: Write integration tests → verify FAIL
3. **Green**: Implement → run tests → verify PASS
4. **Refactor**: Improve code → run tests after each change
5. **Complete**: Done only when ALL tests pass

### Integration Testing Requirements

Constitution Testing Principles I-IX:

- **Integration tests ONLY** (NO mocking), real PostgreSQL via testcontainers
- **Table-driven** with `name` fields
- **Edge cases MANDATORY**: Input validation, boundaries, auth, data state, database, HTTP
- **ServeHTTP testing** via root mux (NOT individual handlers)
- **Protobuf** structs with `protocmp` assertions
- **Derive from fixtures** (NOT response, except UUIDs/timestamps)
- **Run tests** after EVERY change (Principle VI)
- **Map scenarios** to tests (US#-AS#, Principle VIII)
- **Coverage >80%** (Principle IX)

### Error Handling Strategy

Constitution Principle XI:

- **Service**: Sentinel errors (`var ErrXxx`), wrap with `fmt.Errorf("%w")`
- **HTTP**: Error singleton with `ServiceErr` mapping, automatic via `HandleServiceError()`
- **Testing**: ALL errors must have test cases

### Test Database Isolation

- **testcontainers-go** with PostgreSQL (Docker required)
- **Truncation**: `defer truncateTables(db, "tables...")` with CASCADE
- **Parallel**: `t.Parallel()` safe

### Context-Aware Operations

Constitution Principle XII: Services accept `context.Context`, handlers use `r.Context()`, database uses `db.WithContext(ctx)`, tests verify cancellation.

### Distributed Tracing

Constitution Principle XI: HTTP endpoints create OpenTracing spans, services create child spans, database as ONE span per transaction (NOT per query). Dev uses `NoopTracer{}`.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
