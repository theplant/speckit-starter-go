---

description: "Task list template for feature implementation"
---

# Tasks: [FEATURE NAME]

**Input**: Design documents from `/specs/[###-feature-name]/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Testing Approach**: Per Go Project Constitution, this project follows **Test-First Development (TDD)**. Tests are MANDATORY and MUST be written BEFORE implementation. Each task phase includes protobuf definition → test writing → implementation workflow.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story. Each user story follows the TDD cycle completely.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Go Project Structure

All paths reference the Go project structure per plan.md:

- **Protobuf**: `api/proto/v1/` (definitions), `api/gen/v1/` (generated)
- **Services**: `services/` (PUBLIC package - business logic)
- **Handlers**: `handlers/` (PUBLIC package - HTTP layer with integration tests)
- **Models**: `internal/models/` (GORM models - internal only)
- **Tests**: `handlers/*_test.go` (integration tests via ServeHTTP)
- **Fixtures**: `testutil/` (test helpers and fixtures)

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

**Purpose**: Project initialization and basic structure per Constitution Technology Stack

- [ ] T001 Initialize Go module with Go 1.25+
- [ ] T002 [P] Setup `api/proto/v1/` directory for protobuf definitions
- [ ] T003 [P] Setup `api/gen/v1/` directory for generated code
- [ ] T004 [P] Configure `protoc` with `protoc-gen-go`, `protoc-gen-go-grpc`, `protoc-gen-validate`
- [ ] T005 [P] Setup `services/` (public package for business logic)
- [ ] T006 [P] Setup `handlers/` (HTTP handlers)
- [ ] T007 [P] Setup `internal/models/` (GORM models)
- [ ] T008 [P] Setup `testutil/` (test helpers and fixtures)
- [ ] T009 [P] Setup `cmd/api/` (main application entry point)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

**Constitution Requirements**:

- [ ] T010 Setup testcontainers-go with PostgreSQL module in `testutil/db.go`
- [ ] T011 Create `truncateTables()` helper function in `testutil/db.go`
- [ ] T012 Create `setupTestDB()` function with GORM AutoMigrate in `testutil/db.go`
- [ ] T013 [P] Setup error handling: sentinel errors in `services/errors.go`
- [ ] T014 [P] Setup error handling: HTTP error code singleton in `handlers/error_codes.go`
- [ ] T015 [P] Create `HandleServiceError()` with automatic error mapping in `handlers/error_codes.go`
- [ ] T016 [P] Setup OpenTracing with NoopTracer for development
- [ ] T017 [P] Setup HTTP routing structure in `handlers/routes.go` using `http.ServeMux`
- [ ] T018 [P] Setup middleware: logging in `internal/middleware/logging.go`
- [ ] T019 [P] Setup middleware: tracing in `internal/middleware/tracing.go`
- [ ] T020 Create `services/migrations.go` with `AutoMigrate()` function (REQUIRED for external apps)

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - [Title] (Priority: P1) 🎯 MVP

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

**Acceptance Scenarios** (from spec.md):
- US1-AS1: [Scenario description]
- US1-AS2: [Scenario description]

### Step 1: Protobuf Definitions (TDD Phase - Design) 📝

> **CRITICAL**: Define API contracts BEFORE writing tests or code

- [ ] T030 [P] [US1] Define request message in `api/proto/v1/[entity]_service.proto`
- [ ] T031 [P] [US1] Define response message in `api/proto/v1/[entity]_service.proto`
- [ ] T032 [P] [US1] Add protobuf validation rules using `protoc-gen-validate`
- [ ] T033 [US1] Run `protoc` to generate Go code in `api/gen/v1/`
- [ ] T034 [US1] Commit generated protobuf code

### Step 2: Integration Tests (TDD Phase - Red) 🔴

> **CRITICAL**: Write tests FIRST, verify they FAIL before implementation

- [ ] T035 [P] [US1] Create test fixtures in `testutil/fixtures.go` (e.g., `CreateTest[Entity]()`)
- [ ] T036 [US1] Write table-driven test for **US1-AS1** in `handlers/[entity]_handler_test.go`
- [ ] T037 [US1] Write table-driven test for **US1-AS2** in `handlers/[entity]_handler_test.go`
- [ ] T038 [US1] Add edge case tests: **Input validation** (empty strings, nil, SQL injection, XSS)
- [ ] T039 [US1] Add edge case tests: **Boundary conditions** (zero, negative, max values)
- [ ] T040 [US1] Add edge case tests: **Data state** (404, conflicts, duplicates)
- [ ] T041 [US1] Add edge case tests: **Database errors** (constraints, foreign keys)
- [ ] T042 [US1] Add edge case tests: **HTTP specifics** (wrong methods, headers, malformed JSON)
- [ ] T043 [US1] **RUN TESTS** - Verify all tests FAIL (red phase) ❌
- [ ] T044 [US1] Review test design with team/lead before implementation

### Step 3: Implementation (TDD Phase - Green) 🟢

> **CRITICAL**: Write minimal code to make tests pass

- [ ] T045 [P] [US1] Create GORM model in `internal/models/[entity].go`
- [ ] T046 [P] [US1] Add sentinel errors to `services/errors.go` (e.g., `ErrEntityNotFound`)
- [ ] T047 [P] [US1] Add HTTP error codes to `handlers/error_codes.go` with `ServiceErr` mapping
- [ ] T048 [US1] Implement service interface in `services/[entity]_service.go` (returns protobuf types)
- [ ] T049 [US1] Implement service constructor with dependency injection
- [ ] T050 [US1] Implement service methods (e.g., `Create()`, `Get()`, `List()`) with context
- [ ] T051 [US1] Wrap errors with `fmt.Errorf("operation: %w", err)` in service
- [ ] T052 [US1] Add OpenTracing spans to service methods
- [ ] T053 [US1] Create HTTP handler in `handlers/[entity]_handler.go` (thin wrapper)
- [ ] T054 [US1] Add HTTP path patterns to `handlers/routes.go` (e.g., `"POST /api/v1/[entity]"`)
- [ ] T055 [US1] Extract path parameters using `r.PathValue()` in handlers
- [ ] T056 [US1] Add OpenTracing spans to HTTP handlers
- [ ] T057 [US1] Update `services/migrations.go` AutoMigrate() with new model
- [ ] T058 [US1] **RUN TESTS** - Execute full test suite ✅
- [ ] T059 [US1] **VERIFY** - Confirm all tests pass (green phase) ✅

### Step 4: Refactor (TDD Phase - Refactor) ♻️

> **CRITICAL**: Improve code quality while keeping tests green

- [ ] T060 [US1] Refactor: Extract common logic to helper functions
- [ ] T061 [US1] Refactor: Improve error messages and context
- [ ] T062 [US1] Refactor: Add code comments explaining WHY (not WHAT)
- [ ] T063 [US1] **RUN TESTS** - After each refactoring change ✅
- [ ] T064 [US1] **RUN WITH RACE DETECTOR** - `go test -race ./...` ✅

### Step 5: Coverage & Verification ✅

> **CRITICAL**: Ensure comprehensive test coverage

- [ ] T065 [US1] Run `go test -cover ./handlers` - verify >80% coverage
- [ ] T066 [US1] Analyze uncovered paths - add missing tests or remove dead code
- [ ] T067 [US1] Verify ALL sentinel errors have test cases
- [ ] T068 [US1] Verify ALL HTTP error codes have test cases
- [ ] T069 [US1] Verify ALL acceptance scenarios (US1-AS1, US1-AS2) tested
- [ ] T070 [US1] Verify ALL edge case categories covered in tests

**Checkpoint**: User Story 1 fully functional, all tests pass, >80% coverage

---

## Phase 4: User Story 2 - [Title] (Priority: P2)

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

**Acceptance Scenarios** (from spec.md):
- US2-AS1: [Scenario description]
- US2-AS2: [Scenario description]

### Follow TDD Cycle (Same as User Story 1)

> **Pattern**: Protobuf → Tests (Red) → Implementation (Green) → Refactor → Verify

- [ ] T080 [US2] **Step 1**: Define protobuf messages in `api/proto/v1/`
- [ ] T081 [US2] **Step 1**: Generate Go code with `protoc`
- [ ] T082 [US2] **Step 2**: Create test fixtures in `testutil/fixtures.go`
- [ ] T083 [US2] **Step 2**: Write integration tests for US2-AS1, US2-AS2 in `handlers/[entity]_handler_test.go`
- [ ] T084 [US2] **Step 2**: Write ALL edge case tests (6 categories)
- [ ] T085 [US2] **Step 2**: RUN TESTS - Verify FAIL (red) ❌
- [ ] T086 [US2] **Step 3**: Create GORM model in `internal/models/`
- [ ] T087 [US2] **Step 3**: Add sentinel errors and HTTP error codes
- [ ] T088 [US2] **Step 3**: Implement service in `services/` (returns protobuf)
- [ ] T089 [US2] **Step 3**: Implement handler in `handlers/` (thin wrapper)
- [ ] T090 [US2] **Step 3**: Update routes in `handlers/routes.go`
- [ ] T091 [US2] **Step 3**: Update `services/migrations.go`
- [ ] T092 [US2] **Step 3**: RUN TESTS - Verify PASS (green) ✅
- [ ] T093 [US2] **Step 4**: Refactor and run tests after each change ✅
- [ ] T094 [US2] **Step 5**: Run coverage analysis and verify >80%
- [ ] T095 [US2] **Step 5**: Verify ALL scenarios and edge cases tested

**Checkpoint**: User Stories 1 AND 2 both work independently, all tests pass

---

## Phase 5: User Story 3 - [Title] (Priority: P3)

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

**Acceptance Scenarios** (from spec.md):
- US3-AS1: [Scenario description]
- US3-AS2: [Scenario description]

### Follow TDD Cycle (Same Pattern)

> **Pattern**: Protobuf → Tests (Red) → Implementation (Green) → Refactor → Verify

- [ ] T100 [US3] **Step 1**: Define protobuf messages and generate code
- [ ] T101 [US3] **Step 2**: Create fixtures and write integration tests (US3-AS1, US3-AS2)
- [ ] T102 [US3] **Step 2**: Write ALL edge case tests (6 categories)
- [ ] T103 [US3] **Step 2**: RUN TESTS - Verify FAIL (red) ❌
- [ ] T104 [US3] **Step 3**: Implement model, service, handler
- [ ] T105 [US3] **Step 3**: Add errors, routes, migrations
- [ ] T106 [US3] **Step 3**: RUN TESTS - Verify PASS (green) ✅
- [ ] T107 [US3] **Step 4**: Refactor with continuous testing ✅
- [ ] T108 [US3] **Step 5**: Coverage analysis and verification

**Checkpoint**: All user stories (1, 2, 3) independently functional, all tests pass

---

[Add more user story phases as needed, following the same pattern]

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] T200 [P] Run `go test -cover ./...` across all packages - verify >80% coverage
- [ ] T201 [P] Run `go test -race ./...` - verify no race conditions
- [ ] T202 [P] Run `go vet ./...` - verify no issues
- [ ] T203 [P] Run linter (e.g., golangci-lint) - fix all issues
- [ ] T204 [P] Review all sentinel errors have test cases
- [ ] T205 [P] Review all HTTP error codes have test cases
- [ ] T206 [P] Review all acceptance scenarios (US#-AS#) tested
- [ ] T207 Code cleanup: Remove temporary files and helper scripts
- [ ] T208 Code cleanup: Ensure comments explain WHY (not WHAT)
- [ ] T209 Verify services in public `services/` package (NOT internal)
- [ ] T210 Verify services return protobuf types (NOT models)
- [ ] T211 Verify `AutoMigrate()` exported in `services/migrations.go`
- [ ] T212 Performance: Add database indexes if needed
- [ ] T213 Security: Review for SQL injection prevention (handled by GORM)
- [ ] T214 Security: Review for XSS prevention
- [ ] T215 Run quickstart.md validation
- [ ] T216 Update README with setup instructions

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

### Within Each User Story (TDD Cycle)

**CRITICAL - Constitution Principle VII**: Tests MUST be written BEFORE implementation

1. **Design Phase**: Define protobuf messages → Generate Go code
2. **Red Phase**: Write integration tests → Verify they FAIL ❌
3. **Green Phase**: Implement code to make tests pass ✅
4. **Refactor Phase**: Improve code → Run tests after each change ✅
5. **Verify Phase**: Coverage analysis → Verify completeness

**Execution Order**:
- Protobuf definitions before tests
- Tests before models
- Models before services
- Services before handlers
- Handlers before routes
- Run tests after EVERY code change
- Story complete ONLY when all tests pass

### Parallel Opportunities

- All Setup tasks marked [P] can run in parallel
- All Foundational tasks marked [P] can run in parallel (within Phase 2)
- Once Foundational phase completes, all user stories can start in parallel (if team capacity allows)
- All tests for a user story marked [P] can run in parallel
- Models within a story marked [P] can run in parallel
- Different user stories can be worked on in parallel by different team members

---

## Parallel Example: User Story 1 (TDD Cycle)

```bash
# Step 1: Protobuf definitions (can be parallel if different message files)
Task T030: "Define request message in api/proto/v1/product_service.proto"
Task T031: "Define response message in api/proto/v1/product_service.proto"
# Then run: protoc --go_out=api/gen --go-grpc_out=api/gen api/proto/v1/*.proto

# Step 2: Write ALL tests first (can be parallel - different test functions)
Task T036: "Write test for US1-AS1 in handlers/product_handler_test.go"
Task T037: "Write test for US1-AS2 in handlers/product_handler_test.go"
Task T038: "Write edge case tests: input validation"
Task T039: "Write edge case tests: boundary conditions"
# Then run: go test ./handlers -v (should FAIL - red phase)

# Step 3: Implementation (sequential dependencies)
Task T045: "Create GORM model in internal/models/product.go"
Task T046: "Add sentinel errors to services/errors.go"
Task T048: "Implement service in services/product_service.go" (depends on model)
Task T053: "Implement handler in handlers/product_handler.go" (depends on service)
# Then run: go test ./handlers -v (should PASS - green phase)

# Step 4: Refactor (run tests after EACH change)
# Run: go test ./handlers -v (after each refactoring)
```

---

## Implementation Strategy

### MVP First (User Story 1 Only) - TDD Approach

1. Complete Phase 1: Setup (Go project initialization)
2. Complete Phase 2: Foundational (testcontainers, errors, routing) - BLOCKS all stories
3. Complete Phase 3: User Story 1
   - Define protobuf → Generate code
   - Write tests (US1-AS1, US1-AS2 + edge cases) → Verify FAIL ❌
   - Implement (models, service, handler) → Run tests → Verify PASS ✅
   - Refactor → Run tests after each change ✅
   - Coverage analysis → Verify >80%
4. **STOP and VALIDATE**: All User Story 1 tests pass independently
5. Deploy/demo if ready

### Incremental Delivery (TDD Each Story)

1. Setup + Foundational → Foundation ready (testcontainers working)
2. User Story 1 (TDD Cycle):
   - Protobuf → Tests (red) → Implementation (green) → Refactor → Verify
   - **CHECKPOINT**: All tests pass, >80% coverage → Deploy/Demo (MVP!)
3. User Story 2 (TDD Cycle):
   - Protobuf → Tests (red) → Implementation (green) → Refactor → Verify
   - **CHECKPOINT**: All tests pass, US1 still works → Deploy/Demo
4. User Story 3 (TDD Cycle):
   - Protobuf → Tests (red) → Implementation (green) → Refactor → Verify
   - **CHECKPOINT**: All tests pass, US1 & US2 still work → Deploy/Demo
5. Each story adds value without breaking previous stories (verified by tests)

### Parallel Team Strategy (TDD Coordination)

With multiple developers:

1. **Team completes Setup + Foundational together** (MUST be complete first)
2. **Once Foundational is done**, each developer follows TDD cycle:
   - Developer A: User Story 1 (Protobuf → Tests → Implement → Verify)
   - Developer B: User Story 2 (Protobuf → Tests → Implement → Verify)
   - Developer C: User Story 3 (Protobuf → Tests → Implement → Verify)
3. **Coordination Points**:
   - Share protobuf definitions in `api/proto/v1/` (merge conflicts possible)
   - Share test fixtures in `testutil/fixtures.go` (coordinate)
   - Independent handlers in `handlers/` (no conflicts)
   - Run full test suite before merging each story
4. Stories complete independently, all tests pass before integration

---

## Notes

**Task Conventions**:
- [P] tasks = different files, no dependencies, can run in parallel
- [Story] label (US1, US2, US3) maps task to specific user story for traceability
- Task IDs are sequential for easy reference in code reviews

**TDD Requirements (Constitution Principle VII)**:
- Tests MUST be written BEFORE implementation (mandatory, not optional)
- Tests MUST use protobuf structs (NOT `map[string]interface{}`)
- Tests MUST use table-driven design with `name` field
- Tests MUST reference acceptance scenarios (US1-AS1, US1-AS2, etc.)
- Tests MUST cover ALL edge case categories (6 categories, non-negotiable)
- Tests MUST use `cmp.Diff()` with `protocmp.Transform()` for protobuf assertions
- Tests MUST derive expected values from fixtures (NOT response data)
- Tests MUST be run after EVERY code change
- Task is complete ONLY when all tests pass

**Story Independence**:
- Each user story should be independently completable and testable
- Each story has its own protobuf messages, models, services, handlers
- Stories can share foundation (testcontainers, errors, routing)
- Stop at any checkpoint to validate story independently
- All tests for a story must pass before moving to next priority

**Code Organization**:
- Services in public `services/` package (returns protobuf types)
- Handlers in public `handlers/` package (with `*_test.go` integration tests)
- Models in `internal/models/` (GORM models, internal only)
- Test fixtures in `testutil/fixtures.go`
- Protobuf in `api/proto/v1/`, generated in `api/gen/v1/`

**Git Workflow**:
- Commit generated protobuf code after each generation
- Commit after completing each TDD phase (red, green, refactor)
- Run full test suite before committing
- Never commit failing tests or broken code

**Avoid**:
- Vague tasks without specific file paths
- Same file conflicts (coordinate protobuf and fixtures)
- Cross-story dependencies that break independence
- Implementing before writing tests
- Skipping edge case coverage
- Removing tests to make tests pass
