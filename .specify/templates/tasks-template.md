---

description: "Task list template for feature implementation"
---

# Tasks: [FEATURE NAME]

**Input**: Design documents from `/specs/[###-feature-name]/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: Integration tests are MANDATORY per constitution. All tests use real PostgreSQL database via testcontainers-go (no mocking), follow table-driven patterns, use GORM for fixtures, use protobuf structs (NOT maps), verify OpenTracing instrumentation, and cover comprehensive edge cases. Tests are conducted at HTTP layer only (httptest), which exercises the full stack: HTTP → Service → Repository → Database. **Test assertions MUST derive expected values from fixtures** (request data, database fixtures, config), NOT from response data. Only truly random fields (UUIDs, timestamps, crypto/rand) may use response values (Constitution v1.4.3).

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Go project**: Root level for `main.go`, packages in subdirectories, `*_test.go` files alongside source
- **Test organization**: Integration tests in `*_test.go` files (no separate `tests/` directory per Go convention)
- **Test database**: Use testcontainers-go for automatic PostgreSQL container management
- **Architecture**: Three-layer architecture (HTTP → Service → Repository)
- **Service layer**: Business logic in Go interfaces with dependency injection
- **Database access**: Use GORM for all database operations (models, queries, migrations)
- **HTTP framework**: Use standard net/http with http.ServeMux (NO external routers, thin wrappers only)
- **Tracing**: Use OpenTracing for all endpoint instrumentation
- Paths shown below assume Go project structure - adjust based on plan.md

<!-- 
  ============================================================================
  IMPORTANT: The tasks below are SAMPLE TASKS for illustration purposes only.
  
  The /speckit.tasks command MUST replace these with actual tasks based on:
  - User stories from spec.md (with their priorities P1, P2, P3...)
  - Feature requirements from plan.md
  - Entities from data-model.md
  - Endpoints from contracts/
  
  Tasks MUST be organized by user story so each story can be:
  - Implemented independently
  - Tested independently
  - Delivered as an MVP increment
  
  DO NOT keep these sample tasks in the generated tasks.md file.
  ============================================================================
-->

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Initialize Go module with `go mod init`
- [ ] T002 [P] Install GORM dependencies: `go get -u gorm.io/gorm gorm.io/driver/postgres`
- [ ] T003 [P] Install OpenTracing dependency: `go get -u github.com/opentracing/opentracing-go`
- [ ] T004 [P] Install Protocol Buffers dependencies: `go get -u google.golang.org/protobuf/testing/protocmp`
- [ ] T005 [P] Install testcontainers-go: `go get -u github.com/testcontainers/testcontainers-go github.com/testcontainers/testcontainers-go/modules/postgres`
- [ ] T006 [P] Setup GORM connection and database configuration
- [ ] T007 [P] Configure environment variables for database URLs
- [ ] T008 [P] Setup GORM AutoMigrate for database migrations
- [ ] T009 [P] Configure OpenTracing global tracer (NoopTracer for tests, Jaeger/Zipkin for production)
- [ ] T010 [P] Create type-safe error definitions singleton struct (error codes, messages, HTTP status)
- [ ] T011 [P] Configure linting (golangci-lint) and formatting (gofmt, goimports)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T012 Create initial GORM models for core tables
- [ ] T013 Create initial database migrations using GORM AutoMigrate
- [ ] T014 [P] Setup HTTP router using standard net/http with http.ServeMux
- [ ] T015 [P] Implement OpenTracing middleware to instrument all HTTP endpoints
- [ ] T016 [P] Implement middleware: logging, recovery, CORS
- [ ] T017 [P] Create GORM database connection pool and health check
- [ ] T018 [P] Setup testcontainers test database helper (automatic container lifecycle, database truncation for test isolation)
- [ ] T019 Implement base error response types and JSON marshaling (use singleton error struct)
- [ ] T020 [P] Create fixture helper utilities for test database population using GORM
- [ ] T021 [P] Create table truncation helper function (truncate tables in reverse dependency order with CASCADE)
- [ ] T022 [P] Verify OpenTracing spans are created for test requests

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - [Title] (Priority: P1) 🎯 MVP

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

### Integration Tests for User Story 1 (MANDATORY) ⚠️

> **CRITICAL: Write these tests FIRST, ensure they FAIL before implementation**
> **All tests MUST use real PostgreSQL (Docker), table-driven pattern, and cover edge cases**
> **ACCEPTANCE SCENARIO COVERAGE (Principle IX): Each acceptance scenario from spec.md MUST have a corresponding test**

- [ ] T022 [US1] HTTP integration test for [endpoint] in [package]/[handler]_test.go
  - **Acceptance Scenario Tests (MANDATORY - table-driven design per Principle IX)**:
    - Test function: `TestFeature[UserStory]AcceptanceScenarios` (e.g., `TestEnrollmentAcceptanceScenarios`)
    - Use table-driven test with test case struct (Principle II - MANDATORY)
    - Test case `name` field: "US1-AS1: [Description]", "US1-AS2: [Description]", etc.
    - Test case `scenario` field (optional): "Given..., When..., Then..." for documentation
    - Each test case represents one acceptance scenario from spec.md
    - Each test case MUST validate complete "Given/When/Then" clause from spec
    - Use `cmp.Diff()` with `protocmp.Transform()` for ALL protobuf assertions (Principle VI - MANDATORY)
  - Happy path test cases (from acceptance scenarios)
  - Edge cases: input validation, boundary conditions, auth errors
  - Edge cases: data state (404, conflicts), database errors, HTTP specifics
  - Use httptest.ResponseRecorder and real testcontainers PostgreSQL database fixtures
  - Tests exercise full stack: HTTP → Service → Repository → Database
  - Use GORM for fixture data setup
  - Use database truncation for cleanup (defer truncateTables pattern)
  - Use protobuf structs (NOT maps) for request/response
  - Use `cmp.Diff()` with `protocmp.Transform()` for ALL protobuf message assertions (MANDATORY)
  - Do NOT use individual field comparisons for protobuf messages
  - **Build expected from fixtures** (request data, DB fixtures, config), NOT response data (Principle VI v1.4.3)
  - **Read `testutil/fixtures.go`** before writing assertions to identify default values (MANDATORY)
  - **Only use response values** for truly random fields: UUIDs, timestamps, crypto/rand (Constitution v1.4.3)
  - Verify OpenTracing spans are created (NoopTracer default, mock tracer for span verification tests)
  - Verify context.Context is passed through all layers (HTTP → Service → Repository)
  - Verify errors are wrapped with contextual information using `fmt.Errorf("%w", err)`
  - Verify error checking uses `errors.Is()` and `errors.As()` (NOT string comparison)
  - Table-driven test structure with test case structs
  
- [ ] T023 [US1] Create acceptance scenario traceability matrix (optional but recommended)
  - Document mapping: US1-AS1 → test case "US1-AS1: [Description]" in TestFeature[UserStory]AcceptanceScenarios
  - Verify all scenarios from spec.md are covered
  - Flag any deferred scenarios with justification
  - Note: Traceability can also be verified by reviewing test case `name` fields in table-driven test

### Implementation for User Story 1

- [ ] T023 [P] [US1] Create [Entity1] GORM model in [package]/[entity1].go
- [ ] T024 [P] [US1] Create [Entity2] GORM model in [package]/[entity2].go
- [ ] T025 [US1] Implement GORM database repository in [package]/[repository].go (depends on T023, T024)
- [ ] T026 [P] [US1] Define service interface in [package]/service.go with business logic methods (all methods MUST accept context.Context as first parameter)
- [ ] T027 [US1] Implement service with dependency injection (db, logger, cache) in [package]/service_impl.go
- [ ] T028 [US1] Implement business logic in service methods (validation, transactions, error wrapping with fmt.Errorf %w verb)
- [ ] T029 [US1] Implement ServeHTTP handler as thin wrapper in [package]/handler.go (delegates to service)
- [ ] T030 [US1] Add OpenTracing span creation in handler (extract/start span, set tags)
- [ ] T031 [US1] Add child spans for service calls (instrument business logic)
- [ ] T032 [US1] Add HTTP request parsing and response formatting in handler
- [ ] T033 [US1] Create fixture helpers using GORM in [package]/fixtures_test.go
- [ ] T034 [US1] Create service constructor with dependency injection

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently

---

## Phase 4: User Story 2 - [Title] (Priority: P2)

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

### Integration Tests for User Story 2 (MANDATORY) ⚠️

> **ACCEPTANCE SCENARIO COVERAGE (Principle IX): Each acceptance scenario from spec.md MUST have a corresponding test**

- [ ] T018 [US2] Integration test for [endpoint] in [package]/[handler]_test.go
  - **Acceptance Scenario Tests (MANDATORY - table-driven design per Principle IX)**:
    - Test function: `TestFeature[UserStory]AcceptanceScenarios` (table-driven)
    - Test case `name` field: "US2-AS1: [Description]", "US2-AS2: [Description]"
    - Use `cmp.Diff()` with `protocmp.Transform()` for protobuf assertions (MANDATORY)
    - Build expected from fixtures (request, DB, config), NOT response - Constitution v1.4.3
    - Read `testutil/fixtures.go` to identify default values before writing assertions
  - Table-driven tests with comprehensive edge cases per constitution
  
- [ ] T019 [US2] Create acceptance scenario traceability matrix (optional but recommended)
  - Document mapping: US2-AS# → test case name in TestFeature[UserStory]AcceptanceScenarios
  - Verify all scenarios from spec.md are covered

### Implementation for User Story 2

- [ ] T019 [P] [US2] Create [Entity] model/struct in [package]/[entity].go
- [ ] T020 [US2] Implement database operations in [package]/[repository].go
- [ ] T021 [US2] Implement ServeHTTP handler in [package]/[handler].go
- [ ] T022 [US2] Integrate with User Story 1 components (if needed)
- [ ] T023 [US2] Create fixture helpers for test data

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently

---

## Phase 5: User Story 3 - [Title] (Priority: P3)

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

### Integration Tests for User Story 3 (MANDATORY) ⚠️

> **ACCEPTANCE SCENARIO COVERAGE (Principle IX): Each acceptance scenario from spec.md MUST have a corresponding test**

- [ ] T024 [US3] Integration test for [endpoint] in [package]/[handler]_test.go
  - **Acceptance Scenario Tests (MANDATORY - table-driven design per Principle IX)**:
    - Test function: `TestFeature[UserStory]AcceptanceScenarios` (table-driven)
    - Test case `name` field: "US3-AS1: [Description]", "US3-AS2: [Description]"
    - Use `cmp.Diff()` with `protocmp.Transform()` for protobuf assertions (MANDATORY)
  - Table-driven tests with comprehensive edge cases per constitution
  
- [ ] T025 [US3] Create acceptance scenario traceability matrix (optional but recommended)
  - Document mapping: US3-AS# → test case name in TestFeature[UserStory]AcceptanceScenarios
  - Verify all scenarios from spec.md are covered

### Implementation for User Story 3

- [ ] T025 [P] [US3] Create [Entity] model/struct in [package]/[entity].go
- [ ] T026 [US3] Implement database operations in [package]/[repository].go
- [ ] T027 [US3] Implement ServeHTTP handler in [package]/[handler].go
- [ ] T028 [US3] Create fixture helpers for test data

**Checkpoint**: All user stories should now be independently functional

---

[Add more user story phases as needed, following the same pattern]

---

## Phase N-2: Acceptance Scenario Validation (MANDATORY - Before Feature Complete)

**Purpose**: Verify ALL acceptance scenarios from spec.md have corresponding tests per Constitution Principle XIII

**⚠️ CRITICAL**: Feature is NOT complete until all acceptance scenarios are tested. Untested scenarios = untested requirements.

### Acceptance Scenario Validation Tasks (MANDATORY)

- [ ] TXXX Generate acceptance scenario traceability matrix (optional but recommended)
  - List all acceptance scenarios from spec.md (US#-AS# format)
  - Map each scenario to test case name in table-driven tests
  - Example format:
    ```
    | Scenario | Description | Test Function & Case Name | Status |
    |----------|-------------|---------------------------|--------|
    | US1-AS1  | New customer enrolls | TestEnrollmentAcceptanceScenarios → "US1-AS1: New customer enrolls" | ✅ Tested |
    | US1-AS2  | First purchase enrollment | TestEnrollmentAcceptanceScenarios → "US1-AS2: First purchase enrollment" | ❌ NOT TESTED |
    ```

- [ ] TXXX Validate acceptance scenario coverage
  - Verify every scenario in spec.md has a test case in table-driven tests
  - Run tests: `go test -v -run "Test.*AcceptanceScenarios" ./...`
  - Review test output to see test case names (should show "US#-AS#: ..." format)
  - Confirm all tests pass
  - Document any deferred scenarios with justification

- [ ] TXXX Review test structure and assertions
  - Verify table-driven test design used (Principle II - MANDATORY)
  - Verify test case `name` field includes scenario ID: "US#-AS#: [Description]"
  - Verify `cmp.Diff()` with `protocmp.Transform()` used for protobuf assertions (Principle VI - MANDATORY)
  - Verify tests validate complete "Given/When/Then" clauses

**Checkpoint**: All acceptance scenarios tested - requirements validated

---

## Phase N-1: Comprehensive Error Testing (MANDATORY - Before Feature Complete)

**Purpose**: Test ALL defined errors per Constitution Principle IX

**⚠️ CRITICAL**: Feature is NOT complete until all errors are tested. Untested error paths are production bugs.

### Error Testing Tasks (MANDATORY)

- [ ] TXXX Create comprehensive error testing file: `tests/integration/error_handling_test.go`

- [ ] TXXX Implement `TestAllSentinelErrors` function
  - Test EVERY error defined in `services/errors.go`
  - Verify error wrapping with `fmt.Errorf("%w", err)`
  - Verify error checking with `errors.Is()`
  - Use table-driven test structure
  - Each error must have at least one test case
  - Use real database fixtures to trigger errors
  - Example: ErrProductNotFound, ErrDuplicateSKU, ErrMissingRequired, etc.

- [ ] TXXX Implement `TestAllHTTPErrorCodes` function
  - Test EVERY error code defined in `handlers/error_codes.go`
  - Verify HTTP status codes (400, 404, 409, 500, etc.)
  - Verify error response JSON structure
  - Verify ErrorCode.ServiceErr mapping
  - Use httptest.ResponseRecorder for HTTP testing
  - Example: PRODUCT_NOT_FOUND, DUPLICATE_SKU, MISSING_REQUIRED, etc.

- [ ] TXXX Implement `TestErrorFlowEndToEnd` function
  - Verify complete error flow: Service → Handler → Client
  - Test context errors (cancellation → 499, timeout → 504)
  - Validate error message propagation
  - Verify automatic error mapping via HandleServiceError()

- [ ] TXXX Run comprehensive error test suite
  - Execute: `go test -v -run "TestAll.*Errors|TestErrorFlow" ./tests/integration/`
  - Verify every sentinel error has a passing test
  - Verify every HTTP error code has a passing test
  - Confirm zero untested error paths remain

- [ ] TXXX Document error testing approach
  - Create ERROR_TESTING_REPORT.md with coverage matrix
  - List all tested errors with test case names
  - Document any intentionally untested errors (with justification)

**Checkpoint**: All errors tested - ready for code review

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

**Note**: AI agents MUST run full test suite after ALL code changes per Principle XI (Continuous Test Verification)

**Root Cause Tracing** (Principle VIII): When encountering failures, trace backward to find original trigger and fix at source

- [ ] TXXX [P] Documentation updates in docs/
- [ ] TXXX Code cleanup and refactoring (run tests after each change)
- [ ] TXXX Performance optimization across all stories (run tests after each change)
- [ ] TXXX Verify all integration tests pass with real database
- [ ] TXXX Verify all error tests pass
- [ ] TXXX Run tests with race detector to catch concurrency issues
- [ ] TXXX Generate coverage report: `go test -coverprofile=coverage.out ./...`
- [ ] TXXX Analyze coverage gaps and add tests per Principle X
- [ ] TXXX Verify test coverage meets threshold (>80%)
- [ ] TXXX Security hardening (run tests after security changes)
- [ ] TXXX Run quickstart.md validation

**Debugging Discipline** (if issues encountered during any phase):
- [ ] TXXX Document root cause analysis for any bugs fixed
- [ ] TXXX Verify fixes address root causes, not symptoms
- [ ] TXXX Ensure no test cases removed or weakened to make tests pass
- [ ] TXXX Verify test expectations reflect correct behavior
- [ ] TXXX Document tracing process (symptom → source → fix)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3+)**: All depend on Foundational phase completion
  - User stories can then proceed in parallel (if staffed)
  - Or sequentially in priority order (P1 → P2 → P3)
- **Polish (Final Phase)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P2)**: Can start after Foundational (Phase 2) - May integrate with US1 but should be independently testable
- **User Story 3 (P3)**: Can start after Foundational (Phase 2) - May integrate with US1/US2 but should be independently testable

### Within Each User Story

- Tests (if included) MUST be written and FAIL before implementation
- Models before services
- Services before endpoints
- Core implementation before integration
- Story complete before moving to next priority

### Parallel Opportunities

- All Setup tasks marked [P] can run in parallel
- All Foundational tasks marked [P] can run in parallel (within Phase 2)
- Once Foundational phase completes, all user stories can start in parallel (if team capacity allows)
- All tests for a user story marked [P] can run in parallel
- Models within a story marked [P] can run in parallel
- Different user stories can be worked on in parallel by different team members

---

## Parallel Example: User Story 1

```bash
# Launch all tests for User Story 1 together (if tests requested):
Task: "Contract test for [endpoint] in tests/contract/test_[name].py"
Task: "Integration test for [user journey] in tests/integration/test_[name].py"

# Launch all models for User Story 1 together:
Task: "Create [Entity1] model in src/models/[entity1].py"
Task: "Create [Entity2] model in src/models/[entity2].py"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: Test User Story 1 independently
5. Deploy/demo if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 → Test independently → Deploy/Demo (MVP!)
3. Add User Story 2 → Test independently → Deploy/Demo
4. Add User Story 3 → Test independently → Deploy/Demo
5. Each story adds value without breaking previous stories

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: User Story 1
   - Developer B: User Story 2
   - Developer C: User Story 3
3. Stories complete and integrate independently

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Verify tests fail before implementing
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence

---

## Troubleshooting & Debugging (Principle VIII: Root Cause Tracing)

When encountering problems during implementation, follow this methodology:

### Root Cause Tracing Process

**Step 1: Document the Symptom**
- What is the observable problem?
- What behavior was expected vs. actual?
- Where does the problem manifest (test failure, runtime error, etc.)?

**Step 2: Trace Backward Through Call Chain**
```
[SYMPTOM] Observable problem
    ↑
[Layer N] Where problem appears
    ↑
[Layer N-1] Previous call
    ↑
[Layer N-2] Earlier call
    ↑
[ROOT CAUSE] Where problem originates
```

**Step 3: Verify Root Cause**
- Does fixing this source eliminate the symptom?
- Is this the original trigger, not just another symptom?
- Can you explain the mechanism causing the problem?

**Step 4: Fix at Source**
- Implement fix at root cause location
- Do NOT add workarounds in symptom location
- Do NOT weaken tests to accommodate bugs
- Do NOT remove failing test cases

**Step 5: Verify Fix**
- Run tests to confirm problem resolved
- Verify no new problems introduced
- Confirm all tests pass without workarounds

**Step 6: Document**
- Record root cause in commit message
- Add comments explaining the fix if non-obvious
- Update documentation if revealing systemic issue

### Anti-Patterns to Avoid

**❌ NEVER Do These**:
1. Remove failing test cases
2. Change test expectations to match broken behavior
3. Add `t.Skip()` to flaky tests
4. Relax assertions ("at least" instead of "exactly")
5. Add conditional workarounds
6. Catch and ignore errors
7. "Make it work" without understanding why

**✅ ALWAYS Do These**:
1. Trace problem to its source
2. Fix where it originates
3. Maintain test integrity
4. Document root cause
5. Verify proper fix with tests

### Example: Database Default Override

**Symptom**: Struct field `IsActive: false` becomes `true` in database

**Wrong Approach** ❌:
```go
// Remove test case for inactive items
// OR change test: "if len(items) < 2" instead of "== 2"
```

**Root Cause Trace** ✅:
```
Test creates: IsActive: false
    ↑
GORM executes: db.Create(&item)
    ↑
PostgreSQL has: is_active BOOL DEFAULT true
    ↑
ROOT CAUSE: Model tag has `default:true`
```

**Proper Fix** ✅:
```go
// internal/models/item.go
IsActive bool `gorm:"not null;index"` // Removed default:true
```

### When to Apply Root Cause Tracing

Apply this discipline when:
- ✅ Tests fail unexpectedly
- ✅ Behavior doesn't match expectations
- ✅ Errors occur without clear cause
- ✅ Workarounds seem necessary
- ✅ "It should work" but doesn't
- ✅ Flaky tests appear
- ✅ Data doesn't persist as expected

### Documentation Requirements

When fixing bugs, document in commit:
```
Fix: [Brief description]

Root Cause:
- Symptom: [What was observed]
- Traced: [Call chain]
- Source: [Where it originated]
- Fix: [What was changed at source]

Verified: [How fix was confirmed]
```
