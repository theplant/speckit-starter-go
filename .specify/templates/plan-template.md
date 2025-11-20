# Implementation Plan: [FEATURE]

**Branch**: `[###-feature-name]` | **Date**: [DATE] | **Spec**: [link]
**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

[Extract from feature spec: primary requirement + technical approach from research]

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Language/Version**: Go 1.21+ (latest stable recommended)  
**HTTP Framework**: Standard library `net/http` with `http.ServeMux` (MANDATORY per constitution - NO external routers)  
**Database**: PostgreSQL 15+ with JSONB support (MANDATORY per constitution)  
**Database Access**: GORM (gorm.io/gorm with gorm.io/driver/postgres) (MANDATORY per constitution)  
**Distributed Tracing**: OpenTracing (github.com/opentracing/opentracing-go) (MANDATORY per constitution)  
**Protocol Buffers**: protoc compiler, protoc-gen-go for API contracts (MANDATORY per constitution)  
**Testing**: Standard library `testing` with `httptest`, testcontainers-go for PostgreSQL (MANDATORY per constitution)  
**Test Comparison**: google/go-cmp with protocmp for protobuf assertions (MANDATORY per constitution)  
**Error Handling**: Standard library fmt.Errorf with %w for wrapping, errors.Is/As for checking (MANDATORY per constitution)  
**Error Testing**: ALL sentinel errors and HTTP error codes MUST be tested (MANDATORY per constitution Principle IX)  
**Context Propagation**: All service methods MUST accept context.Context as first parameter (MANDATORY per constitution)  
**Service Architecture**: Services in public `services/` package (NOT internal/) for external reusability (MANDATORY per constitution Principle VIII)  
**Target Platform**: [e.g., Linux server, containerized deployment, Kubernetes or NEEDS CLARIFICATION]  
**Project Type**: Backend API service (Go)  
**Performance Goals**: [domain-specific, e.g., 1000 req/s, p99 < 200ms or NEEDS CLARIFICATION]  
**Constraints**: [domain-specific, e.g., <200ms p95, <100MB memory or NEEDS CLARIFICATION]  
**Scale/Scope**: [domain-specific, e.g., 10k users, 100k requests/day or NEEDS CLARIFICATION]

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
  for this feature. Expand the structure with real paths for your API project.
-->

```text
# Go Backend API Project Structure
├── .specify/              # Spec-kit configuration
│   ├── memory/
│   │   └── constitution.md
│   ├── scripts/
│   └── templates/
├── services/              # PUBLIC - reusable business logic
│   ├── product_service.go
│   ├── errors.go          # Sentinel errors
│   └── migrations.go      # AutoMigrate for external apps
├── handlers/              # PUBLIC - HTTP handlers
│   ├── product_handler.go
│   └── error_codes.go
├── api/                   # PUBLIC - protobuf definitions and generated code
│   ├── v1/
│   │   └── product.proto
│   └── gen/v1/
│       └── product.pb.go
├── internal/              # INTERNAL - implementation details
│   ├── models/            # GORM models (not exposed)
│   ├── middleware/        # HTTP middleware
│   └── config/            # Configuration
├── cmd/                   # Application entry points
│   └── api/
│       └── main.go
└── tests/
    └── integration/
        └── product_test.go
```

**Structure Decision**: [Document any customizations to the structure above for your specific API needs]

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
