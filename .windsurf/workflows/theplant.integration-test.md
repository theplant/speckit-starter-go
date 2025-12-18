---
description: Write integration tests for Go microservices - real database, no mocking, test through root mux ServeHTTP.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Write integration tests following Principle INTEGRATION_TESTING - use real PostgreSQL database, no mocking, test through root mux ServeHTTP.

## Principle INTEGRATION_TESTING: Integration Testing (No Mocking)

- Tests MUST use real PostgreSQL database connections (NO mocking)
- Test database MUST be isolated per test run with table truncation
- Fixtures MUST be inserted via GORM and represent realistic production data
- Implementation code MUST NEVER contain mocks or mock interfaces
- Dependencies requiring testability MUST be injected via interfaces using builder pattern (Principle SERVICE_ARCHITECTURE)
- Test code MUST NOT call `internal/` packages for setup - dependencies MUST be injected via service constructors
- **Exception (models-only fixtures)**: `testutil/` and fixture helpers MAY import `internal/models` strictly for inserting realistic database fixtures via GORM
- Mocking is ONLY permitted in test code when testing interactions with external systems (third-party APIs, message queues)
- Mock implementations MUST be defined in `*_test.go` files (NEVER in production code files)
- Any use of mocks in tests MUST include written justification explaining why integration testing is infeasible

**Rationale**: Integration tests catch real-world issues mocks cannot (constraints, transactions, serialization). Real fixtures validate actual database behavior. Mocks in implementation code create fake abstraction layers that hide real behavior.

## Execution Steps

### 1. Design API Contract

Define API contract (OpenAPI or Protobuf) → generate Go code:
- For OpenAPI: Create/update `api/openapi/<domain>.yaml`
- For Protobuf: Create/update `api/proto/<domain>/v1/<domain>.proto`

// turbo
```bash
go generate ./...
```

### 2. Setup Test Database

Use testcontainers for automatic PostgreSQL lifecycle:

```go
func setupTestDB(t *testing.T) (*gorm.DB, func()) {
    ctx := context.Background()
    
    pgContainer, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:15-alpine"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("postgres"),
        postgres.WithPassword("postgres"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2),
        ),
    )
    if err != nil {
        t.Fatalf("Failed to start PostgreSQL container: %v", err)
    }
    
    cleanup := func() {
        pgContainer.Terminate(ctx)
    }
    
    connStr, _ := pgContainer.ConnectionString(ctx, "sslmode=disable")
    db, _ := gorm.Open(postgres.Open(connStr), &gorm.Config{})
    db.AutoMigrate(&models.Product{} /* add models */)
    
    return db, cleanup
}

func truncateTables(db *gorm.DB, tables ...string) {
    for i := len(tables) - 1; i >= 0; i-- {
        db.Exec(fmt.Sprintf("TRUNCATE TABLE %s CASCADE", tables[i]))
    }
}
```

### 3. Write Table-Driven Tests (Principle TABLE_DRIVEN)

Test cases MUST be defined as slices of structs with descriptive `name` fields:

```go
import (
    "bytes"
    "io"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/google/go-cmp/cmp"
    "google.golang.org/protobuf/encoding/protojson"
    "google.golang.org/protobuf/testing/protocmp"
)

func TestProductAPI(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    // Setup service and router using builder pattern
    service := services.NewProductService(db).Build()
    mux := handlers.NewRouter(service, db).Build()
    
    testCases := []struct {
        name          string
        method        string
        path          string
        body          *pb.CreateProductRequest
        setupFixtures func()
        wantStatus    int
        wantProduct   *pb.Product       // For success responses
        wantErr       *pb.ErrorResponse // For error responses (full struct, not just code)
    }{
        {
            name:       "US1-AS1: create product with valid data",
            method:     "POST",
            path:       "/api/v1/products",
            body:       &pb.CreateProductRequest{Name: "Test Product", Sku: "SKU-001"},
            wantStatus: http.StatusCreated,
            wantProduct: &pb.Product{
                Name: "Test Product",
                Sku:  "SKU-001",
            },
        },
        {
            name:       "US1-AS2: create product with empty name returns validation error",
            method:     "POST",
            path:       "/api/v1/products",
            body:       &pb.CreateProductRequest{Name: "", Sku: "SKU-001"},
            wantStatus: http.StatusBadRequest,
            wantErr: &pb.ErrorResponse{
                Code:    "MISSING_REQUIRED",
                Message: "Required field missing",
            },
        },
        {
            name:   "US1-AS3: create product with duplicate SKU returns conflict",
            method: "POST",
            path:   "/api/v1/products",
            body:   &pb.CreateProductRequest{Name: "Test", Sku: "DUP-001"},
            setupFixtures: func() {
                db.Create(&models.Product{Name: "Existing", SKU: "DUP-001"})
            },
            wantStatus: http.StatusConflict,
            wantErr: &pb.ErrorResponse{
                Code:    "DUPLICATE_SKU",
                Message: "SKU already exists",
            },
        },
    }
    
    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            truncateTables(db, "products")
            if tc.setupFixtures != nil {
                tc.setupFixtures()
            }
            
            var body io.Reader
            if tc.body != nil {
                data, _ := protojson.Marshal(tc.body)
                body = bytes.NewReader(data)
            }
            
            req := httptest.NewRequest(tc.method, tc.path, body)
            rec := httptest.NewRecorder()
            
            // Test through root mux ServeHTTP (Principle SERVEHTTP_TESTING)
            mux.ServeHTTP(rec, req)
            
            // Compare status code
            if rec.Code != tc.wantStatus {
                t.Fatalf("status = %d, want %d, body: %s", rec.Code, tc.wantStatus, rec.Body.String())
            }
            
            // Compare full response struct using cmp.Diff (Principle SCHEMA_FIRST)
            if tc.wantProduct != nil {
                var actual pb.Product
                if err := protojson.Unmarshal(rec.Body.Bytes(), &actual); err != nil {
                    t.Fatalf("failed to unmarshal response: %v", err)
                }
                
                // Use actual values for generated fields (ID, timestamps)
                expected := &pb.Product{
                    Id:        actual.Id,        // Generated UUID
                    Name:      tc.wantProduct.Name,
                    Sku:       tc.wantProduct.Sku,
                    CreatedAt: actual.CreatedAt, // Generated timestamp
                    UpdatedAt: actual.UpdatedAt, // Generated timestamp
                }
                
                if diff := cmp.Diff(expected, &actual, protocmp.Transform()); diff != "" {
                    t.Errorf("Product mismatch (-want +got):\n%s", diff)
                }
            }
            
            if tc.wantErr != nil {
                var actual pb.ErrorResponse
                if err := protojson.Unmarshal(rec.Body.Bytes(), &actual); err != nil {
                    t.Fatalf("failed to unmarshal error response: %v", err)
                }
                
                // Compare full error response struct
                if diff := cmp.Diff(tc.wantErr, &actual, protocmp.Transform()); diff != "" {
                    t.Errorf("Error response mismatch (-want +got):\n%s", diff)
                }
            }
        })
    }
}
```

### 4. Run Tests

// turbo
```bash
go test -v ./...
```

### 5. Run Tests with Race Detector

// turbo
```bash
go test -v -race ./...
```

### 6. Verify Tests FAIL Before Implementation

- Tests MUST fail before implementation code is written
- This confirms the test is actually testing the right thing

### 7. Implement Minimal Code

- Write minimal code to make tests pass
- Services MUST use builder pattern: `NewService(db).Build()`
- ALL validation MUST be in services (handlers are thin wrappers)

### 8. Refactor

- Improve code while keeping tests green
- Run tests after each change (Principle CONTINUOUS_TESTING)

## Why Root Mux ServeHTTP (Principle SERVEHTTP_TESTING)

Calling `handler.Create(rec, req)` bypasses routing, middleware, and method matching. Tests pass even with broken route registration (e.g., typo in path).

```go
// ❌ WRONG: Bypasses routing
handler.Create(rec, req)

// ✅ CORRECT: Tests complete HTTP stack
mux.ServeHTTP(rec, req)
```

## Struct Comparison (Principle SCHEMA_FIRST)

Tests MUST compare structs using `cmp.Diff()` with `protocmp.Transform()`. NEVER compare individual fields.

### Why Struct Comparison Over Field Comparison

| Benefit | Field Comparison | Struct Comparison |
|---------|------------------|-------------------|
| **New fields** | Tests pass silently when new fields are added but not set | Fails immediately, forcing you to update test |
| **Removed fields** | Tests still pass even if field is removed | Fails immediately, catching breaking changes |
| **Error messages** | "name mismatch" - no context | Full diff showing exactly what differs |
| **Maintenance** | Must add assertion for every new field | Automatically covers all fields |
| **Refactoring** | Easy to forget to update assertions | Catches all structural changes |
| **Nested structs** | Must manually recurse into nested objects | Automatically compares nested structures |

### The Silent Failure Problem

```go
// ❌ WRONG: Field comparison - SILENT FAILURES
func TestProduct_FieldComparison(t *testing.T) {
    actual := getProduct()
    
    // If API adds a new field "Description", this test still passes!
    // If API removes "SKU" field, this test still passes!
    // If "Price" is wrong, you won't know until production!
    if actual.Name != "Test Product" {
        t.Error("name mismatch")
    }
    if actual.SKU != "SKU-001" {
        t.Error("sku mismatch")
    }
    // Forgot to check Price, Description, CreatedAt, UpdatedAt...
}

// ✅ CORRECT: Struct comparison - CATCHES EVERYTHING
func TestProduct_StructComparison(t *testing.T) {
    actual := getProduct()
    
    expected := &pb.Product{
        Id:          actual.Id,  // Use actual for generated fields
        Name:        "Test Product",
        Sku:         "SKU-001",
        Price:       1999,
        Description: "A test product",
        CreatedAt:   actual.CreatedAt,  // Use actual for timestamps
    }
    
    if diff := cmp.Diff(expected, actual, protocmp.Transform()); diff != "" {
        t.Errorf("Product mismatch (-want +got):\n%s", diff)
    }
}
```

### Handling Dynamic Fields (IDs, Timestamps)

For fields that are generated (UUIDs, timestamps), use the actual value in expected:

```go
expected := &pb.Product{
    Id:        actual.Id,        // ✅ Generated UUID - use actual
    Name:      "Test Product",   // ✅ From request - use expected value
    Sku:       "SKU-001",        // ✅ From request - use expected value
    CreatedAt: actual.CreatedAt, // ✅ Generated timestamp - use actual
    UpdatedAt: actual.UpdatedAt, // ✅ Generated timestamp - use actual
}
```

### Comparing Lists/Slices

```go
// ✅ CORRECT: Compare entire list
expected := &pb.ListProductsResponse{
    Products: []*pb.Product{
        {Id: products[0].Id, Name: "Product A", Sku: "SKU-A"},
        {Id: products[1].Id, Name: "Product B", Sku: "SKU-B"},
    },
    Total: 2,
}

if diff := cmp.Diff(expected, actual, protocmp.Transform()); diff != "" {
    t.Errorf("List mismatch (-want +got):\n%s", diff)
}

// ❌ WRONG: Only checking length
if len(actual.Products) != 2 {
    t.Error("wrong count")
}
```

### Ignoring Specific Fields

When you need to ignore certain fields (e.g., always-changing timestamps):

```go
import "github.com/google/go-cmp/cmp/cmpopts"

// Ignore specific fields
opts := cmp.Options{
    protocmp.Transform(),
    cmpopts.IgnoreFields(&pb.Product{}, "CreatedAt", "UpdatedAt"),
}

if diff := cmp.Diff(expected, actual, opts...); diff != "" {
    t.Errorf("Mismatch (-want +got):\n%s", diff)
}
```

### Error Response Comparison

```go
// ✅ CORRECT: Compare full error response
var errResp pb.ErrorResponse
protojson.Unmarshal(rec.Body.Bytes(), &errResp)

expected := &pb.ErrorResponse{
    Code:    "MISSING_REQUIRED",
    Message: "Required field missing",
}

if diff := cmp.Diff(expected, &errResp, protocmp.Transform()); diff != "" {
    t.Errorf("Error response mismatch (-want +got):\n%s", diff)
}

// ❌ WRONG: Only checking code
if errResp.Code != "MISSING_REQUIRED" {
    t.Error("wrong error code")
}
// Message could be wrong and you'd never know!
```

### Required Imports

```go
import (
    "github.com/google/go-cmp/cmp"
    "github.com/google/go-cmp/cmp/cmpopts"  // For IgnoreFields
    "google.golang.org/protobuf/testing/protocmp"
)
```

### Measures to Enforce Struct Comparison

1. **Code Review Checklist**: Reject PRs with individual field assertions
2. **Linter Rule**: Flag `if actual.Field != expected.Field` patterns
3. **Test Template**: Always start with `cmp.Diff()` in test templates
4. **CI Check**: Grep for field comparison patterns in test files

```bash
# CI check: Find field comparisons in tests (should return empty)
grep -rn "if.*actual\.\w* !=" *_test.go && exit 1 || exit 0
```

### Summary: NEVER Do This

```go
// ❌ NEVER compare individual fields
if actual.Name != expected.Name { t.Error("name") }
if actual.SKU != expected.SKU { t.Error("sku") }
if actual.Price != expected.Price { t.Error("price") }

// ❌ NEVER use reflect.DeepEqual (no diff output)
if !reflect.DeepEqual(expected, actual) { t.Error("mismatch") }

// ❌ NEVER use == for structs (doesn't work for pointers/slices)
if expected == actual { t.Error("mismatch") }
```

### Summary: ALWAYS Do This

```go
// ✅ ALWAYS use cmp.Diff with protocmp.Transform
if diff := cmp.Diff(expected, actual, protocmp.Transform()); diff != "" {
    t.Errorf("Mismatch (-want +got):\n%s", diff)
}
```

## AI Agent Requirements

- MUST use real PostgreSQL via testcontainers (NO mocking)
- MUST test through root mux ServeHTTP (NOT individual handlers)
- MUST use table-driven tests with descriptive names
- MUST use generated structs and `cmp.Diff()` for comparison
- MUST verify tests FAIL before implementation
- MUST run tests after EVERY code change

## See Also

- `/theplant.testdb` - Test database setup with testcontainers
- `/theplant.service` - Service layer with builder pattern
- `/theplant.routes` - HTTP routing setup
- `/theplant.errors` - Error handling strategy
