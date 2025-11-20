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
**Target Platform**: [e.g., Linux server, containerized deployment or NEEDS CLARIFICATION]  
**Project Type**: [single/web/mobile - determines source structure]  
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
  for this feature. Delete unused options and expand the chosen structure with
  real paths (e.g., apps/admin, packages/something). The delivered plan must
  not include Option labels.
-->

```text
# [REMOVE IF UNUSED] Option 1: Single project (DEFAULT)
src/
├── models/
├── services/
├── cli/
└── lib/

tests/
├── contract/
├── integration/
└── unit/

# [REMOVE IF UNUSED] Option 2: Web application (when "frontend" + "backend" detected)
backend/
├── src/
│   ├── models/
│   ├── services/
│   └── api/
└── tests/

frontend/
├── src/
│   ├── components/
│   ├── pages/
│   └── services/
└── tests/

# [REMOVE IF UNUSED] Option 3: Mobile + API (when "iOS/Android" detected)
api/
└── [same as backend above]

ios/ or android/
└── [platform-specific structure: feature modules, UI flows, platform tests]
```

**Structure Decision**: [Document the selected structure and reference the real
directories captured above]

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
