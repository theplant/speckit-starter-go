---

description: "Task list template for feature implementation"
---

# Tasks: [FEATURE NAME]

**Input**: Design documents from `/specs/[###-feature-name]/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Testing**: TDD mandatory - write tests BEFORE implementation (protobuf → tests → implementation).
**Organization**: Tasks grouped by user story for independent implementation.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Go Project Structure

- **OpenAPI**: `api/openapi/` (SINGLE SOURCE OF TRUTH for API contracts)
- **Protobuf**: `api/proto/v1/` (GENERATED from OpenAPI - DO NOT HAND-EDIT), `api/gen/v1/` (generated Go code)
- **Services**: `services/` (PUBLIC - business logic, returns protobuf)
- **Handlers**: `handlers/` (PUBLIC - HTTP layer with `*_test.go`, includes `helpers.go`, `middleware.go`, `routes.go`)
- **Models**: `internal/models/` (GORM - internal only)
- **Fixtures**: `testutil/` (helpers and fixtures - MAY import `internal/models` for fixtures)

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
- [ ] T003 [P] Configure `openapi2proto` and `protoc` with `protoc-gen-go`, `protoc-gen-go-grpc`
- [ ] T004 [P] Setup `services/`, `handlers/`, `internal/models/`, `testutil/`, `cmd/api/`

---

## Phase 2: Foundation (⚠️ BLOCKS all user stories)

- [ ] T010 Setup testcontainers-go in `testutil/db.go` (`setupTestDB()`, `truncateTables()`)
- [ ] T011 [P] Setup error handling in `services/errors.go` and `handlers/error_codes.go`
- [ ] T012 [P] Create `HandleServiceError()`, `RespondWithError()`, `RespondWithProto()`, `DecodeProtoJSON()` in `handlers/helpers.go`
- [ ] T013 [P] Setup OpenTracing NoopTracer and routing builder in `handlers/routes.go`
- [ ] T014 [P] Setup middleware in `handlers/middleware.go`
- [ ] T015 Create `services/migrations.go` with `AutoMigrate()` function

---

## Phase 3: User Story 1 - [Title] (P1) 🎯 MVP

**Goal**: [Brief description]  
**Acceptance Scenarios**: US1-AS1, US1-AS2

### Step 1: OpenAPI → Protobuf (Design) 📝

- [ ] T030 [P] [US1] Define API contract in `api/openapi/[domain].json` with `x-proto-tag` for field numbers
- [ ] T031 [US1] Generate `.proto` via `openapi2proto`, run `protoc`, commit generated code (NEVER hand-edit .proto)

### Step 2: Tests (Red) 🔴

- [ ] T035 [P] [US1] Create test fixtures in `testutil/fixtures.go`
- [ ] T036 [US1] Write tests for US1-AS1, US1-AS2 in `handlers/[entity]_handler_test.go`
- [ ] T037 [US1] Add ALL edge case tests (input validation, boundaries, data state, database, HTTP)
- [ ] T038 [US1] **RUN TESTS** - Verify FAIL (red) ❌

### Step 3: Implementation (Green) 🟢

- [ ] T045 [P] [US1] Create model in `internal/models/`, errors in `services/errors.go`, HTTP codes in `handlers/error_codes.go`
- [ ] T046 [US1] Implement service in `services/[entity]_service.go` (returns protobuf, uses context)
- [ ] T047 [US1] Implement handler in `handlers/[entity]_handler.go` (thin wrapper)
- [ ] T048 [US1] Add routes to `handlers/routes.go`, update `services/migrations.go`
- [ ] T049 [US1] Add OpenTracing spans to handler and service
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

- [ ] T080 [US2] OpenAPI → Protobuf → Generate code (NEVER hand-edit .proto)
- [ ] T081 [US2] Write tests (US2-AS1, US2-AS2 + edge cases) → Verify FAIL ❌
- [ ] T082 [US2] Implement (model, errors, service, handler, routes) → RUN TESTS → Verify PASS ✅
- [ ] T083 [US2] Refactor → Run tests after each change ✅
- [ ] T084 [US2] Coverage analysis → Verify >80%

---

## Phase 5: User Story 3 - [Title] (P3)

**Goal**: [Brief description]  
**Acceptance Scenarios**: US3-AS1, US3-AS2

- [ ] T100 [US3] OpenAPI → Protobuf → Tests → Implement → Refactor → Verify (same TDD cycle)

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
- OpenAPI → Protobuf (generated) → Tests (red) → Implementation (green) → Refactor → Verify
- Tests BEFORE implementation (mandatory)
- Run tests after EVERY code change
- Story complete ONLY when all tests pass

**Parallel**: Tasks marked [P] can run in parallel

## Implementation Strategy

**MVP First** (Story 1 only): Setup → Foundation → US1 (TDD) → Validate → Deploy

**Incremental**: Each story follows TDD cycle (Protobuf → Red → Green → Refactor → Verify), checkpoint after each

**Parallel Team**: Complete Foundation together, then stories in parallel (coordinate on protobuf/fixtures)

---

## Notes

**Conventions**:
- [P] = parallel, [US#] = user story mapping
- Tests BEFORE implementation (TDD mandatory, Principle VI)
- Story complete ONLY when all tests pass

**Code Organization**:
- Services/handlers: PUBLIC packages (return protobuf)
- Models: `internal/models/` (GORM, internal only)
- OpenAPI: `api/openapi/` (SINGLE SOURCE OF TRUTH)
- Protobuf: `api/proto/v1/` (GENERATED from OpenAPI - DO NOT HAND-EDIT), `api/gen/v1/` (generated Go code)

**Testing Requirements** (Constitution Principles I-IX):
- **I. Integration Testing**: Real PostgreSQL via testcontainers (NO mocking in implementation code). Mocking ONLY in test code (`*_test.go`) for external systems with justification. Test setup uses public APIs and dependency injection (NOT direct `internal/` package imports). Exception: `testutil/` MAY import `internal/models` strictly for fixtures.
- **II. Table-Driven Design**: Test cases as slices of structs with descriptive `name` fields, execute using `t.Run(testCase.name, func(t *testing.T) {...})`
- **III. Edge Cases MANDATORY**: Input validation (empty/nil, invalid formats, SQL injection, XSS), boundaries (zero/negative/max), auth (missing/expired/invalid tokens), data state (404s, conflicts), database (constraint violations, foreign key failures), HTTP (wrong methods, missing headers, invalid content-types, malformed JSON)
- **IV. ServeHTTP Testing**: Call root mux ServeHTTP (NOT individual handlers) using `httptest.ResponseRecorder`, identical routing configuration from shared routes package, `r.PathValue()` for parameters
- **V. Protobuf Structs (Generated from OpenAPI)**: API contracts in OpenAPI (single source of truth), `.proto` generated via `openapi2proto` (NEVER hand-edit), stabilize field numbers via `x-proto-tag`. Use `cmp.Diff()` with `protocmp.Transform()`. Derive expected from TEST FIXTURES (request data, database fixtures, config). Copy from response ONLY for truly random: UUIDs, timestamps, crypto-rand tokens. Read `testutil/fixtures.go` for defaults before writing assertions.
- **VI. Continuous Test Verification**: Run `go test -v ./...` and `go test -v -race ./...` after EVERY change. Fix failures immediately (NO skipping/disabling tests).
- **VII. Root Cause Tracing**: Trace backward through call chain, fix source NOT symptoms, NEVER remove/weaken tests
- **VIII. Acceptance Scenario Coverage**: Every US#-AS# has corresponding test with scenario ID in test case name
- **IX. Coverage >80%**: Run `go test -coverprofile=coverage.out ./...`, analyze with `go tool cover -func=coverage.out`

**Architecture Requirements** (Constitution Principles X-XIII):
- **X. Service Layer Architecture**: Business logic in Go interfaces (service layer), services in public packages (NOT `internal/`), handlers thin wrappers. **ALL validation in services** (handlers MUST NOT validate, including empty/nil checks). Builder pattern: `NewService(db).WithLogger(log).Build()`. Routes use builder: `NewRouter(svc, db).WithMiddlewares(...).Build()`. Service methods use protobuf structs for ALL parameters/return types (NO primitives, NO maps). `cmd/main.go` calls ONLY handlers/services (NEVER `internal/`).
- **XI. Distributed Tracing**: HTTP endpoints create OpenTracing spans with operation name (e.g., "POST /api/products"). Services create child spans. Database: ONE span per transaction (NOT per query). External calls propagate trace context. Errors set `span.SetTag("error", true)`. Dev uses `opentracing.NoopTracer{}`.
- **XII. Context-Aware Operations**: Service methods accept `context.Context` as first parameter. Handlers use `r.Context()`. Database uses `db.WithContext(ctx)`. External calls use `http.NewRequestWithContext(ctx, ...)`. Long-running ops check cancellation periodically. Tests verify cancellation behavior.
- **XIII. Error Handling Strategy**: Service Layer defines sentinel errors in `services/errors.go`, wraps with `fmt.Errorf("%w")`. HTTP Layer defines error codes in `handlers/error_codes.go` with `ServiceErr` field. `HandleServiceError()` provides automatic mapping (NO switch statements). **Environment-Aware Details**: ErrorResponse includes `details` field with full error chain by default; hidden when `HIDE_ERROR_DETAILS=true`. `RespondWithError(w, errCode, err)` always passes original error. ALL errors have test cases. Tests use error code definitions (NOT literal strings).

**Avoid**: Implementing before tests, skipping edge cases, removing tests to pass
