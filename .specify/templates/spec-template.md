# Feature Specification: [FEATURE NAME]

**Feature Branch**: `[###-feature-name]`  
**Created**: [DATE]  
**Status**: Draft  
**Input**: User description: "$ARGUMENTS"

## User Scenarios & Testing *(mandatory)*

<!--
  IMPORTANT: User stories should be PRIORITIZED as user journeys ordered by importance.
  Each user story/journey must be INDEPENDENTLY TESTABLE - meaning if you implement just ONE of them,
  you should still have a viable MVP (Minimum Viable Product) that delivers value.
  
  Assign priorities (P1, P2, P3, etc.) to each story, where P1 is the most critical.
  Think of each story as a standalone slice of functionality that can be:
  - Developed independently
  - Tested independently
  - Deployed independently
  - Demonstrated to users independently
  
  CRITICAL - Scenario Identification:
  Each acceptance scenario has a unique ID (e.g., "US1-AS1", "US1-AS2") for tracking.
  These IDs will be used to map scenarios to automated tests during implementation.
  This ensures every scenario listed here is verified in the system (Constitution Principle VIII).
-->

### User Story 1 - [Brief Title] (Priority: P1)

[Describe this user journey in plain language]

**Why this priority**: [Explain the value and why it has this priority level]

**Independent Test**: [Describe how this can be tested independently - e.g., "Can be fully tested by [specific action] and delivers [specific value]"]

**Acceptance Scenarios**:

1. **US1-AS1**: **Given** [initial state], **When** [action], **Then** [expected outcome]
2. **US1-AS2**: **Given** [initial state], **When** [action], **Then** [expected outcome]

---

### User Story 2 - [Brief Title] (Priority: P2)

[Describe this user journey in plain language]

**Why this priority**: [Explain the value and why it has this priority level]

**Independent Test**: [Describe how this can be tested independently]

**Acceptance Scenarios**:

1. **US2-AS1**: **Given** [initial state], **When** [action], **Then** [expected outcome]
2. **US2-AS2**: **Given** [initial state], **When** [action], **Then** [expected outcome]

---

### User Story 3 - [Brief Title] (Priority: P3)

[Describe this user journey in plain language]

**Why this priority**: [Explain the value and why it has this priority level]

**Independent Test**: [Describe how this can be tested independently]

**Acceptance Scenarios**:

1. **US3-AS1**: **Given** [initial state], **When** [action], **Then** [expected outcome]
2. **US3-AS2**: **Given** [initial state], **When** [action], **Then** [expected outcome]

---

[Add more user stories as needed, each with an assigned priority]

### Edge Cases

<!--
  ACTION REQUIRED: Document what happens in exceptional or boundary situations.
  Think about what could go wrong or what unusual inputs/states the system might encounter.
  
  These edge cases will be tested according to Constitution Principle III:
  - Input validation: Empty/nil values, invalid formats, SQL injection, XSS
  - Boundary conditions: Zero/negative/max values, empty arrays, nil pointers
  - Auth: Missing/expired/invalid tokens, insufficient permissions
  - Data state: 404s, conflicts, concurrent modifications
  - Database: Constraint violations, foreign key failures, transaction conflicts
  - HTTP: Wrong methods, missing headers, invalid content-types, malformed JSON
-->

**Invalid or Missing Input**:
- What happens when required information is not provided?
- What happens when information is in the wrong format?
- What happens with malicious input (e.g., script injection attempts)?
- [Add specific scenarios for this feature]

**Boundary Conditions**:
- What happens with zero, negative, or very large numbers?
- What happens with empty lists or missing data?
- [Add specific scenarios for this feature]

**Access Control** (if applicable):
- What happens when users try to access things they shouldn't?
- What happens when credentials are missing or expired?
- [Add specific scenarios for this feature]

**Data Conflicts**:
- What happens when trying to access something that doesn't exist?
- What happens when trying to create something that already exists?
- What happens when two people modify the same thing simultaneously?
- [Add specific scenarios for this feature]

**System Errors**:
- What happens when data can't be saved?
- What happens when related information is missing or invalid?
- [Add specific scenarios for this feature]

## Requirements *(mandatory)*

<!--
  ACTION REQUIRED: The content in this section represents placeholders.
  Fill them out with the right functional requirements.
-->

### Functional Requirements

- **FR-001**: System MUST [specific capability, e.g., "allow users to create accounts"]
- **FR-002**: System MUST [specific capability, e.g., "validate email addresses"]  
- **FR-003**: Users MUST be able to [key interaction, e.g., "reset their password"]
- **FR-004**: System MUST [data requirement, e.g., "persist user preferences"]
- **FR-005**: System MUST [behavior, e.g., "log all security events"]

**Error Handling Requirements** (Constitution Principle XIII):
- **FR-ERR-001**: System MUST provide clear error messages when operations fail
- **FR-ERR-002**: System MUST distinguish between user errors (invalid input) and system errors (technical failures)
- **FR-ERR-003**: Error messages MUST NOT expose sensitive technical details to users

*Example of marking unclear requirements:*

- **FR-006**: System MUST authenticate users via [NEEDS CLARIFICATION: auth method not specified - email/password, SSO, OAuth?]
- **FR-007**: System MUST retain user data for [NEEDS CLARIFICATION: retention period not specified]

### Key Entities *(include if feature involves data)*

- **[Entity 1]**: [What it represents, key attributes without implementation]
- **[Entity 2]**: [What it represents, relationships to other entities]

## Success Criteria *(mandatory)*

<!--
  ACTION REQUIRED: Define measurable success criteria.
  These must be technology-agnostic and measurable.
-->

### Measurable Outcomes

- **SC-001**: [Measurable metric, e.g., "Users can complete account creation in under 2 minutes"]
- **SC-002**: [Measurable metric, e.g., "System handles 1000 concurrent users without degradation"]
- **SC-003**: [User satisfaction metric, e.g., "90% of users successfully complete primary task on first attempt"]
- **SC-004**: [Business metric, e.g., "Reduce support tickets related to [X] by 50%"]

### Verification Requirements

All acceptance scenarios and edge cases listed above MUST be:

- **Testable**: Each scenario can be demonstrated and verified in a test environment
- **Complete**: Tests verify the entire expected behavior, not partial outcomes
- **Automated**: Tests can be run repeatedly without manual intervention
- **Independent**: Each scenario can be tested separately

Every acceptance scenario (US#-AS#) listed above will have a corresponding automated test that validates the expected outcome matches the "Then" clause (Constitution Principle VIII).

### Testing Requirements

Implementation MUST follow Constitution Testing Principles:

- **I. Integration Testing (No Mocking)**: Real PostgreSQL via testcontainers (NO mocking in implementation code), test isolation with table truncation, fixtures via GORM. Mocking ONLY in test code (`*_test.go`) for external systems with written justification. Mock implementations NEVER in production code files.
- **II. Table-Driven Design**: Test cases as slices of structs with descriptive `name` fields, execute using `t.Run(testCase.name, func(t *testing.T) {...})`
- **III. Comprehensive Edge Case Coverage**: All edge cases listed above MUST have corresponding tests (input validation, boundary conditions, auth, data state, database, HTTP)
- **IV. ServeHTTP Endpoint Testing**: Tests call root mux ServeHTTP (NOT individual handlers) using `httptest.ResponseRecorder`, identical routing configuration from shared routes package, HTTP path patterns, `r.PathValue()` for parameters
- **V. Protobuf Data Structures**: API contracts in `.proto` files (single source of truth), tests use protobuf structs (NO `map[string]interface{}`), compare using `cmp.Diff()` with `protocmp.Transform()` (NO `==`, `reflect.DeepEqual`, or individual field checks). Expected values MUST be derived from TEST FIXTURES (request data, database fixtures, config). Copy from response ONLY for truly random fields: UUIDs, timestamps, crypto-rand tokens.
- **VI. Continuous Test Verification**: Tests MUST be executed after EVERY code change. Run `go test -v ./...` and `go test -v -race ./...` for concurrency safety. Tests MUST pass before commit. Test failures MUST be fixed immediately (NO skipping/disabling tests).
- **VII. Root Cause Tracing (Debugging Discipline)**: When problems occur, MUST trace backward through call chain to find root cause. Distinguish symptoms from root causes. Fixes MUST address source, NOT work around symptoms. Test cases MUST NOT be removed or weakened. Use debuggers and logging to understand control flow.
- **VIII. Acceptance Scenario Coverage (Spec-to-Test Mapping)**: Every user scenario (US#-AS#) in this spec MUST have corresponding automated test. Test case names MUST reference source scenarios (e.g., "US1-AS1: New customer enrolls"). Tests MUST validate complete "Then" clause, not partial behavior.
- **IX. Test Coverage & Gap Analysis**: Run `go test -coverprofile=coverage.out ./...` and `go tool cover -func=coverage.out` to identify gaps. Target >80% coverage for business logic. Remove dead code if unreachable.

### Architecture Requirements

Implementation MUST follow Constitution System Architecture Principles:

- **X. Service Layer Architecture (Dependency Injection)**: Business logic MUST be Go interfaces (service layer). Services MUST NOT depend on HTTP types (only `context.Context` allowed). Handlers MUST be thin wrappers delegating to services. Services and handlers MUST be in public packages (NOT `internal/`) for reusability. External dependencies MUST be injected via builder pattern: `NewService(db).WithLogger(log).Build()`. Service methods MUST use protobuf structs for ALL parameters and return types (NO primitives, NO maps). `cmd/main.go` MUST ONLY call handlers or services (NEVER `internal/` packages directly).

- **XI. Distributed Tracing (OpenTracing)**: HTTP endpoints MUST create OpenTracing spans with operation name (e.g., "POST /api/products"). Service methods SHOULD create child spans (e.g., "ProductService.Create"). Database operations: ONE span per transaction (NOT per SQL query - too much overhead). External calls (HTTP, gRPC) MUST propagate trace context. Errors MUST set `span.SetTag("error", true)`. Spans MUST include tags: `http.method`, `http.url`, `http.status_code`. Development/Tests use `opentracing.NoopTracer{}`, Production configured from environment variables.

- **XII. Context-Aware Operations**: Service methods MUST accept `context.Context` as first parameter. HTTP handlers MUST use `r.Context()`. Database operations MUST use `db.WithContext(ctx)`. External HTTP calls MUST use `http.NewRequestWithContext(ctx, ...)`. Long-running operations MUST check context cancellation periodically. Tests MUST verify context cancellation behavior. **Rationale**: Enables timeout handling, graceful cancellation, trace propagation, prevents resource leaks.

- **XIII. Comprehensive Error Handling**: Two-layer strategy with environment-aware details. **Service Layer**: Sentinel errors (package-level vars) with `fmt.Errorf("%w")` wrapping for error breadcrumb trail. **HTTP Layer**: Singleton error code struct with automatic service error mapping via `HandleServiceError()` (NO switch statements). **Environment-Aware Details**: ErrorResponse includes `details` field with full error chain by default; hidden when `HIDE_ERROR_DETAILS=true`. `RespondWithError(w, errCode, err)` always passes original error. **Testing**: ALL errors (sentinel + HTTP codes) MUST have test cases. Error assertions MUST use error code definitions (NOT literal strings).
