---

description: "Task list template for feature implementation"
---

# Tasks: [FEATURE NAME]

**Input**: Design documents from `/specs/[###-feature-name]/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Testing**: TDD mandatory - write tests BEFORE implementation (OpenAPI → ogen → tests → implementation).
**Organization**: Tasks grouped by user story for independent implementation.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Go Project Structure

- **OpenAPI**: `api/openapi/` (API contract definitions - YAML preferred, ErrorResponse MUST be defined here)
- **Generated Code**: `api/gen/<domain>/` (ogen generated Go types and server interfaces)
- **Services**: `services/` (PUBLIC - implements ogen Handler interface)
- **Handlers**: `handlers/` (PUBLIC - server setup with ErrorHandler, error codes)
- **Tests**: `tests/` (SEPARATE package for integration tests with testutil helpers)
- **Models**: `internal/models/` (GORM - internal only)

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

## Phase 1: Setup

- [ ] T001 Initialize Go module with Go 1.25+
- [ ] T002 [P] Setup `api/openapi/`, `api/proto/v1/`, and `api/gen/v1/`
- [ ] T003 [P] Configure `ogen` with go:generate directive
- [ ] T004 [P] Setup `services/`, `handlers/`, `internal/models/`, `testutil/`, `cmd/api/`

---

## Phase 2: Foundation (⚠️ BLOCKS all user stories)

- [ ] T010 Setup testcontainers-go in `tests/testutil_test.go` (`setupTestDB()`, `truncateTables()`)
- [ ] T011 [P] Setup sentinel errors in `services/errors.go`
- [ ] T012 [P] Define ErrorResponse in `api/openapi/<domain>.yaml`, regenerate with ogen
- [ ] T013 [P] Create `handlers/error_codes.go` with ErrorCode struct and Errors singleton
- [ ] T014 [P] Create `handlers/error_handler.go` with `OgenErrorHandler()` and `mapServiceError()`
- [ ] T015 [P] Create `handlers/server.go` with `NewServer()` wrapper using `api.WithErrorHandler()`
- [ ] T016 Create `services/migrations.go` with `AutoMigrate()` function

---

## Phase 3: User Story 1 - [Title] (P1) 🎯 MVP

**Goal**: [Brief description]  
**Acceptance Scenarios**: US1-AS1, US1-AS2

### Step 1: API Contract & Code Generation (Design) 📝

- [ ] T030 [P] [US1] Define API contract in `api/openapi/[domain].yaml`
- [ ] T031 [US1] Generate Go code via `ogen` (`go generate ./...`), commit generated code

### Step 2: Tests (Red) 🔴

- [ ] T035 [P] [US1] Create test fixtures in `tests/testutil_test.go`
- [ ] T036 [US1] Write tests for US1-AS1, US1-AS2 in `tests/[entity]_test.go`
- [ ] T037 [US1] Add ALL edge case tests (input validation, boundaries, data state, database, HTTP)
- [ ] T038 [US1] **RUN TESTS** - Verify FAIL (red) ❌

### Step 3: Implementation (Green) 🟢

- [ ] T045 [P] [US1] Create model in `internal/models/`, errors in `services/errors.go`, HTTP codes in `handlers/error_codes.go`
- [ ] T046 [US1] Implement service in `services/[entity]_service.go` (implements ogen Handler interface)
- [ ] T048 [US1] Update `services/migrations.go` with new models
- [ ] T049 [US1] Add OpenTracing spans to service methods
- [ ] T050 [US1] **RUN TESTS** - Verify PASS (green) ✅

### Step 4: Refactor ♻️

- [ ] T060 [US1] Refactor (extract helpers, improve errors, add comments)
- [ ] T061 [US1] **RUN TESTS** after each change ✅, run with `-race` ✅

### Step 5: Verify ✅

- [ ] T065 [US1] Run `go test -cover` - verify >80% coverage
- [ ] T066 [US1] Verify ALL errors, scenarios, and edge cases tested

---

## Phase 4: User Story 2 - [Title] (P2)

**Goal**: [Brief description]  
**Acceptance Scenarios**: US2-AS1, US2-AS2

### TDD Cycle (Same Pattern)

- [ ] T080 [US2] OpenAPI → ogen → Generate code (`go generate ./...`)
- [ ] T081 [US2] Write tests (US2-AS1, US2-AS2 + edge cases) → Verify FAIL ❌
- [ ] T082 [US2] Implement (model, errors, service, handler, routes) → RUN TESTS → Verify PASS ✅
- [ ] T083 [US2] Refactor → Run tests after each change ✅
- [ ] T084 [US2] Coverage analysis → Verify >80%

---

## Phase 5: User Story 3 - [Title] (P3)

**Goal**: [Brief description]  
**Acceptance Scenarios**: US3-AS1, US3-AS2

- [ ] T100 [US3] OpenAPI → ogen → Tests → Implement → Refactor → Verify (same TDD cycle)

---

[Add more user story phases as needed, following the same pattern]

---

## Phase N: Polish

- [ ] T200 [P] Run `go test -cover ./...` - verify >80%, `go test -race ./...`, `go vet ./...`
- [ ] T201 [P] Verify ALL errors/scenarios tested, services in public packages
- [ ] T202 Code cleanup: Remove temp files, comments explain WHY
- [ ] T203 Security: Review SQL injection/XSS prevention
- [ ] T204 Update README

---

## Execution Order

**Phase Dependencies**:
- Phase 1 (Setup) → Phase 2 (Foundation) → Phase 3+ (User Stories) → Phase N (Polish)
- Foundation BLOCKS all user stories
- User stories can proceed in parallel after Foundation

**TDD Cycle** (Constitution Principle VI):
- OpenAPI → ogen (generated) → Tests (red) → Implementation (green) → Refactor → Verify
- Tests BEFORE implementation (mandatory)
- Run tests after EVERY code change
- Story complete ONLY when all tests pass

**Parallel**: Tasks marked [P] can run in parallel

## Implementation Strategy

**MVP First** (Story 1 only): Setup → Foundation → US1 (TDD) → Validate → Deploy

**Incremental**: Each story follows TDD cycle (ogen → Red → Green → Refactor → Verify), checkpoint after each

**Parallel Team**: Complete Foundation together, then stories in parallel (coordinate on OpenAPI/fixtures)

---

## Notes

**Conventions**:
- [P] = parallel, [US#] = user story mapping
- Tests BEFORE implementation (TDD mandatory, Principle VI)
- Story complete ONLY when all tests pass

**Code Organization**:
- Services: PUBLIC packages (implement ogen Handler interface, return generated types)
- Handlers: PUBLIC `handlers/` package (server setup with ErrorHandler, error codes)
- Models: `internal/models/` (GORM, internal only)
- OpenAPI: `api/openapi/` (API contract definitions, ErrorResponse MUST be defined here)
- Generated: `api/gen/<domain>/` (ogen generated Go types and server interfaces)

**Testing Requirements** (Constitution Principles I-IX):
- **I. Integration Testing**: Real PostgreSQL via testcontainers (NO mocking in implementation code). Mocking ONLY in test code (`*_test.go`) for external systems with justification. Test setup uses public APIs and dependency injection (NOT direct `internal/` package imports). Exception: `testutil/` MAY import `internal/models` strictly for fixtures.
- **II. Table-Driven Design**: Test cases as slices of structs with descriptive `name` fields, execute using `t.Run(testCase.name, func(t *testing.T) {...})`
- **III. Edge Cases MANDATORY**: Input validation (empty/nil, invalid formats, SQL injection, XSS), boundaries (zero/negative/max), auth (missing/expired/invalid tokens), data state (404s, conflicts), database (constraint violations, foreign key failures), HTTP (wrong methods, missing headers, invalid content-types, malformed JSON)
- **IV. ServeHTTP Testing**: Call root mux ServeHTTP (NOT individual handlers) using `httptest.ResponseRecorder`, identical routing configuration from shared routes package, `r.PathValue()` for parameters
- **V. API Data Structures (Schema-First Design)**: OpenAPI with `ogen` - generate Go types and `Handler` interface via `go run github.com/ogen-go/ogen/cmd/ogen@latest`. **Services implement `Handler` in `services/` package (NEVER in handlers/)**. Use `cmp.Diff()` for comparisons. Derive expected from TEST FIXTURES. Copy from response ONLY for truly random: UUIDs, timestamps, crypto-rand tokens.
- **VI. Continuous Test Verification**: Run `go test -v ./...` and `go test -v -race ./...` after EVERY change. Fix failures immediately (NO skipping/disabling tests).
- **VII. Root Cause Tracing**: Trace backward through call chain, fix source NOT symptoms, NEVER remove/weaken tests
- **VIII. Acceptance Scenario Coverage**: Every US#-AS# has corresponding test with scenario ID in test case name
- **IX. Coverage >80%**: Run `go test -coverprofile=coverage.out ./...`, analyze with `go tool cover -func=coverage.out`

**Architecture Requirements** (Constitution Principles X-XIII):
- **X. Service Layer Architecture**: **Services implement ogen-generated `Handler` interface in `services/` package (NEVER in handlers/)**. Services are reusable Go packages with OpenAPI-defined interfaces. **ogen provides built-in validation from OpenAPI schema**. Builder pattern: `NewService(db).WithLogger(log).Build()`. Service methods use ogen-generated structs for ALL parameters/return types (NO primitives, NO maps). `cmd/main.go` uses `handlers.NewServer()` (NOT `api.NewServer()` directly). `handlers/` package provides server setup with ErrorHandler.
- **XI. Distributed Tracing**: HTTP endpoints create OpenTracing spans with operation name (e.g., "POST /api/products"). Services create child spans. Database: ONE span per transaction (NOT per query). External calls propagate trace context. Errors set `span.SetTag("error", true)`. Dev uses `opentracing.NoopTracer{}`.
- **XII. Context-Aware Operations**: Service methods accept `context.Context` as first parameter. Handlers use `r.Context()`. Database uses `db.WithContext(ctx)`. External calls use `http.NewRequestWithContext(ctx, ...)`. Long-running ops check cancellation periodically. Tests verify cancellation behavior.
- **XIII. Error Handling Strategy**: Service Layer defines sentinel errors in `services/errors.go`, wraps with `fmt.Errorf("%w")`. **ErrorResponse MUST be defined in OpenAPI schema** (generated by ogen). `handlers/error_handler.go` implements ogen ErrorHandler, maps service errors via `errors.Is()`. `handlers/server.go` provides `NewServer()` with `api.WithErrorHandler(OgenErrorHandler)`. **Environment-Aware Details**: ErrorResponse includes `details` field with full error chain by default; hidden when `HIDE_ERROR_DETAILS=true`. ALL errors have test cases. Tests use error code definitions (NOT literal strings).

**Avoid**: Implementing before tests, skipping edge cases, removing tests to pass
