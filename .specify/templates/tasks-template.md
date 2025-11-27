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

- **Protobuf**: `api/proto/v1/` (source), `api/gen/v1/` (generated)
- **Services**: `services/` (PUBLIC - business logic, returns protobuf)
- **Handlers**: `handlers/` (PUBLIC - HTTP layer with `*_test.go`)
- **Models**: `internal/models/` (GORM - internal only)
- **Fixtures**: `testutil/` (helpers and fixtures)

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
- [ ] T002 [P] Setup `api/proto/v1/` and `api/gen/v1/`
- [ ] T003 [P] Configure `protoc` with `protoc-gen-go`, `protoc-gen-validate`
- [ ] T004 [P] Setup `services/`, `handlers/`, `internal/models/`, `testutil/`, `cmd/api/`

---

## Phase 2: Foundation (⚠️ BLOCKS all user stories)

- [ ] T010 Setup testcontainers-go in `testutil/db.go` (`setupTestDB()`, `truncateTables()`)
- [ ] T011 [P] Setup error handling in `services/errors.go` and `handlers/error_codes.go`
- [ ] T012 [P] Create `HandleServiceError()` in `handlers/error_codes.go`
- [ ] T013 [P] Setup OpenTracing NoopTracer and routing in `handlers/routes.go`
- [ ] T014 [P] Setup middleware in `internal/middleware/`
- [ ] T015 Create `services/migrations.go` with `AutoMigrate()` function

---

## Phase 3: User Story 1 - [Title] (P1) 🎯 MVP

**Goal**: [Brief description]  
**Acceptance Scenarios**: US1-AS1, US1-AS2

### Step 1: Protobuf (Design) 📝

- [ ] T030 [P] [US1] Define request/response messages in `api/proto/v1/[entity]_service.proto`
- [ ] T031 [US1] Add validation rules, run `protoc`, commit generated code

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

- [ ] T080 [US2] Protobuf → Generate code
- [ ] T081 [US2] Write tests (US2-AS1, US2-AS2 + edge cases) → Verify FAIL ❌
- [ ] T082 [US2] Implement (model, errors, service, handler, routes) → RUN TESTS → Verify PASS ✅
- [ ] T083 [US2] Refactor → Run tests after each change ✅
- [ ] T084 [US2] Coverage analysis → Verify >80%

---

## Phase 5: User Story 3 - [Title] (P3)

**Goal**: [Brief description]  
**Acceptance Scenarios**: US3-AS1, US3-AS2

- [ ] T100 [US3] Protobuf → Tests → Implement → Refactor → Verify (same TDD cycle)

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
- Protobuf → Tests (red) → Implementation (green) → Refactor → Verify
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
- Protobuf: `api/proto/v1/` (source), `api/gen/v1/` (generated)

**Avoid**: Implementing before tests, skipping edge cases, removing tests to pass
