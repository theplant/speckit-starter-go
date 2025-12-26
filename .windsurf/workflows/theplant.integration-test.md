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

## Principle API_STRUCTS: Use ogen API Structs (Never Plain JSON)

**CRITICAL**: Tests MUST use ogen-generated API structs for request bodies. NEVER use plain JSON strings.

### Why API Structs Over Plain JSON

| Problem with Plain JSON | Benefit of API Structs |
|------------------------|------------------------|
| No compile-time type checking | Compiler catches field name typos |
| Easy to miss required fields | Struct enforces required fields |
| JSON field names can drift from API | Struct stays in sync with OpenAPI |
| No IDE autocomplete | Full IDE support |
| Refactoring doesn't update tests | Refactoring updates all usages |

### Helper Function for ogen Types

ogen types have custom `MarshalJSON` methods. Use this helper:

```go
// ogenEncoder is an interface for ogen-generated types that have MarshalJSON
type ogenEncoder interface {
    MarshalJSON() ([]byte, error)
}

// newAPIRequest creates an HTTP request with an ogen API struct as body
func newAPIRequest(t *testing.T, method, path string, body ogenEncoder) *http.Request {
    t.Helper()
    var bodyReader io.Reader
    var contentLength int64

    if body != nil {
        data, err := body.MarshalJSON()
        if err != nil {
            t.Fatalf("Failed to marshal request body: %v", err)
        }
        bodyReader = bytes.NewReader(data)
        contentLength = int64(len(data))
    }

    req := httptest.NewRequest(method, path, bodyReader)
    if body != nil {
        req.Header.Set("Content-Type", "application/json")
        req.ContentLength = contentLength
    }
    return req
}
```

### Example: WRONG vs CORRECT

```go
// ❌ WRONG: Plain JSON string - no type safety
jsonBody := `{"firstName":"John","lastName":"Doe","email":"john@test.com","role":"admin"}`
req := httptest.NewRequest("POST", "/users", strings.NewReader(jsonBody))

// ✅ CORRECT: ogen API struct - type-safe, IDE support
createReq := &api.CreateUserRequest{
    FirstName: "John",
    LastName:  "Doe",
    Email:     "john@test.com",
    Role:      api.UserRoleAdmin,
}
req := newAPIRequest(t, "POST", "/users", createReq)
```

### Table-Driven Tests with API Structs

```go
testCases := []struct {
    name       string
    request    *api.LoginRequest  // ✅ Use API struct type
    wantStatus int
}{
    {
        name: "valid credentials",
        request: &api.LoginRequest{
            Email:    "test@test.com",
            Password: "password123",
        },
        wantStatus: http.StatusOK,
    },
    {
        name: "invalid password",
        request: &api.LoginRequest{
            Email:    "test@test.com",
            Password: "wrongpassword",
        },
        wantStatus: http.StatusUnauthorized,
    },
}

for _, tc := range testCases {
    t.Run(tc.name, func(t *testing.T) {
        req := newAPIRequest(t, "POST", "/auth/login", tc.request)
        // ...
    })
}
```

## Principle COMPLETE_TEST_DATA: Fill All Struct Fields Completely

**CRITICAL**: When creating test input data or expected response structs, you MUST:

1. **Fill ALL fields** - Never leave any field empty, nil, or with zero values unless explicitly testing that scenario
2. **Nested structs MUST be fully populated** - Recursively fill all nested struct fields down to primitive values
3. **Array/slice fields MUST have at least 2 elements** - Never use empty arrays `[]` unless testing empty case
4. **Each array element MUST be fully populated** - Every element in arrays must have all its fields filled
5. **Use realistic production-like values** - Values should represent real-world data, not placeholder text
6. **Pointer fields MUST point to fully populated structs** - No nil pointers unless testing nil case

### Why Complete Test Data Matters

| Problem | Consequence |
|---------|-------------|
| Empty arrays `[]` | Serialization bugs not caught (JSON `[]` vs `null`) |
| Nil nested structs | Nil pointer dereference bugs missed |
| Missing fields | Default value bugs not detected |
| Single-element arrays | Off-by-one and iteration bugs missed |
| Placeholder values | Data validation bugs not caught |

### Example: WRONG vs CORRECT

```go
// ❌ WRONG: Empty arrays, missing nested fields, placeholder values
createInput := pim.ProductCreateInput{
    Sku:         "TEST-001",
    Name:        pim.LocalizedValue{"en-US": "Test"},  // Only 1 locale
    Slug:        "test",
    Status:      pim.Draft,
    Tags:        []string{},           // ❌ Empty array
    CategoryIds: []string{},           // ❌ Empty array
    MediaIds:    []string{},           // ❌ Empty array
    Media:       []pim.ProductMedia{}, // ❌ Empty array
    Attributes:  []pim.ProductAttribute{}, // ❌ Empty array
    Variants:    []pim.ProductVariant{},   // ❌ Empty array
    Pricing:     pim.ProductPricing{Price: 10, Currency: "USD"}, // Missing optional fields
    Seo:         pim.ProductSEO{},     // ❌ Empty nested struct
}

// ✅ CORRECT: All fields filled, arrays with 2+ elements, nested structs complete
createInput := pim.ProductCreateInput{
    Sku:  "LAPTOP-PRO-15-2024",
    Name: pim.LocalizedValue{
        "en-US": "Professional Laptop 15-inch",
        "zh-CN": "专业笔记本电脑15寸",
    },
    Slug:   "professional-laptop-15-inch",
    Status: pim.Draft,
    ShortDescription: pim.LocalizedValue{
        "en-US": "High-performance laptop for professionals",
        "zh-CN": "专业人士高性能笔记本",
    },
    LongDescription: pim.LocalizedValue{
        "en-US": "The Professional Laptop 15-inch features cutting-edge technology...",
        "zh-CN": "专业笔记本电脑15寸采用尖端技术...",
    },
    Tags:        []string{"electronics", "laptop", "professional"},
    CategoryIds: []string{category1.ID, category2.ID},
    MediaIds:    []string{media1.ID, media2.ID},
    Media: []pim.ProductMedia{
        {
            MediaId:  media1.ID,
            Position: 1,
            Alt:      "Laptop front view",
            Role:     "primary",
        },
        {
            MediaId:  media2.ID,
            Position: 2,
            Alt:      "Laptop side view",
            Role:     "gallery",
        },
    },
    Attributes: []pim.ProductAttribute{
        {
            Code:  "screen_size",
            Value: "15.6 inches",
        },
        {
            Code:  "processor",
            Value: "Intel Core i7-12700H",
        },
    },
    Variants: []pim.ProductVariant{
        {
            Sku:    "LAPTOP-PRO-15-2024-8GB",
            Name:   pim.LocalizedValue{"en-US": "8GB RAM Variant", "zh-CN": "8GB内存版本"},
            Status: pim.Active,
            Pricing: pim.ProductPricing{
                Price:         999.99,
                ComparePrice:  1199.99,
                CostPrice:     750.00,
                Currency:      "USD",
            },
            Attributes: []pim.ProductAttribute{
                {Code: "ram", Value: "8GB"},
                {Code: "storage", Value: "256GB SSD"},
            },
        },
        {
            Sku:    "LAPTOP-PRO-15-2024-16GB",
            Name:   pim.LocalizedValue{"en-US": "16GB RAM Variant", "zh-CN": "16GB内存版本"},
            Status: pim.Active,
            Pricing: pim.ProductPricing{
                Price:         1299.99,
                ComparePrice:  1499.99,
                CostPrice:     950.00,
                Currency:      "USD",
            },
            Attributes: []pim.ProductAttribute{
                {Code: "ram", Value: "16GB"},
                {Code: "storage", Value: "512GB SSD"},
            },
        },
    },
    Pricing: pim.ProductPricing{
        Price:        999.99,
        ComparePrice: 1199.99,
        CostPrice:    750.00,
        Currency:     "USD",
    },
    Seo: pim.ProductSEO{
        MetaTitle:       "Professional Laptop 15-inch | Best Performance",
        MetaDescription: "Buy the Professional Laptop 15-inch with Intel Core i7...",
        MetaKeywords:    []string{"laptop", "professional", "intel", "15-inch"},
        CanonicalUrl:    "https://example.com/products/professional-laptop-15-inch",
    },
}
```

### Checklist Before Submitting Test Code

- [ ] Every struct field has a non-zero value (unless testing zero/nil case)
- [ ] Every array/slice has at least 2 elements
- [ ] Every nested struct is fully populated
- [ ] Every array element has all its fields filled
- [ ] Values are realistic (not "test", "foo", "bar")
- [ ] LocalizedValue maps have at least 2 locales

## Principle EXPECTED_FROM_INPUT: Expected Values Must Come From Input, Not Results

**CRITICAL**: When building expected response structs for comparison, you MUST:

1. **NEVER copy values from execution results into expected structs** - This defeats the purpose of testing
2. **Build expected structs entirely from input data** - Expected values should be derived from what you sent
3. **Use `cmpopts.IgnoreFields()` for generated/dynamic fields** - IDs, timestamps, and other server-generated values
4. **Nested structs with generated fields should also use IgnoreFields** - Apply recursively

### Why This Matters

If you copy values from the result into the expected struct, you're essentially saying "I expect the result to equal itself" - which always passes and tests nothing meaningful.

```go
// ❌ WRONG: Copying from result defeats the test
var actual pim.Product
json.Unmarshal(rec.Body.Bytes(), &actual)

expected := pim.Product{
    Id:          actual.Id,          // ❌ Copied from result - always matches
    Sku:         actual.Sku,         // ❌ Copied from result - always matches
    Variants:    actual.Variants,    // ❌ Copied from result - always matches
    CreatedAt:   actual.CreatedAt,   // ❌ Copied from result - always matches
}
// This test ALWAYS passes even if the API is completely broken!

// ✅ CORRECT: Expected from input, ignore generated fields
var actual pim.Product
json.Unmarshal(rec.Body.Bytes(), &actual)

expected := pim.Product{
    Sku:    createInput.Sku,     // ✅ From input - verifies API saved correctly
    Name:   createInput.Name,    // ✅ From input - verifies API saved correctly
    Slug:   createInput.Slug,    // ✅ From input - verifies API saved correctly
    Status: createInput.Status,  // ✅ From input - verifies API saved correctly
    Tags:   createInput.Tags,    // ✅ From input - verifies API saved correctly
    Pricing: pim.ProductPricing{
        Price:    createInput.Pricing.Price,    // ✅ From input
        Currency: createInput.Pricing.Currency, // ✅ From input
    },
    // Don't include generated fields - use IgnoreFields instead
}

// Ignore generated/dynamic fields
opts := cmp.Options{
    cmpopts.IgnoreFields(pim.Product{}, "Id", "CreatedAt", "UpdatedAt"),
    cmpopts.IgnoreFields(pim.ProductVariant{}, "Id"),
    cmpopts.EquateEmpty(),
}

if diff := cmp.Diff(expected, actual, opts...); diff != "" {
    t.Errorf("Product mismatch (-want +got):\n%s", diff)
}
```

### Fields That Should Be Ignored (Not Copied)

| Field Type | Examples | Why Ignore |
|------------|----------|------------|
| Generated IDs | `Id`, `ProductId`, `VariantId` | Server generates UUIDs |
| Timestamps | `CreatedAt`, `UpdatedAt`, `DeletedAt` | Server generates timestamps |
| Computed fields | `TotalCount`, `SyncedCount` | Server computes these |
| Nested struct IDs | `Variants[].Id`, `Media[].Id` | Nested entities also get generated IDs |

### How to Handle Nested Structs with Generated Fields

```go
// For nested structs like Variants, build expected from input and ignore nested IDs
expected := pim.Product{
    Sku:  createInput.Sku,
    Name: createInput.Name,
    Variants: []pim.ProductVariant{
        {
            Sku:      createInput.Variants[0].Sku,      // ✅ From input
            Name:     createInput.Variants[0].Name,     // ✅ From input
            Price:    createInput.Variants[0].Price,    // ✅ From input
            IsActive: createInput.Variants[0].IsActive, // ✅ From input
            // Id is ignored via cmpopts.IgnoreFields
        },
        {
            Sku:      createInput.Variants[1].Sku,
            Name:     createInput.Variants[1].Name,
            Price:    createInput.Variants[1].Price,
            IsActive: createInput.Variants[1].IsActive,
        },
    },
}

opts := cmp.Options{
    cmpopts.IgnoreFields(pim.Product{}, "Id", "CreatedAt", "UpdatedAt"),
    cmpopts.IgnoreFields(pim.ProductVariant{}, "Id"),
    cmpopts.EquateEmpty(),
}
```

## Execution Steps

First, create a task plan using `update_plan`:

```
Call update_plan with:
- explanation: "Integration test workflow for <feature>"
- plan: [
    {"step": "Design API contract (OpenAPI/Protobuf)", "status": "pending"},
    {"step": "Setup test database with testcontainers", "status": "pending"},
    {"step": "Write table-driven tests with API structs", "status": "pending"},
    {"step": "Run tests and verify they fail first", "status": "pending"},
    {"step": "Implement minimal code to pass tests", "status": "pending"},
    {"step": "Run tests with race detector", "status": "pending"},
    {"step": "Refactor while keeping tests green", "status": "pending"}
  ]
```

Then execute each step, updating status to `in_progress` before starting and `completed` after finishing.

### 1. Design API Contract

Define API contract (OpenAPI or Protobuf) → generate Go code:
- For OpenAPI: Create/update `api/openapi/<domain>.yaml`
- For Protobuf: Create/update `api/proto/<domain>/v1/<domain>.proto`

// turbo
```bash
go generate ./...
```

### 2. Setup Test Database

Use testcontainers for automatic PostgreSQL lifecycle.

**Prerequisites:**
- Docker must be available (local/CI environments)
- Install testcontainers-go:
```bash
go get github.com/testcontainers/testcontainers-go
go get github.com/testcontainers/testcontainers-go/modules/postgres
```

```go
import (
    "context"
    "testing"
    
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func setupTestDB(t *testing.T) (*gorm.DB, func()) {
    ctx := context.Background()
    
    // Create PostgreSQL container
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
    
    // Cleanup function
    cleanup := func() {
        if err := pgContainer.Terminate(ctx); err != nil {
            t.Logf("Failed to terminate container: %v", err)
        }
    }
    
    // Get connection string
    connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
        cleanup()
        t.Fatalf("Failed to get connection string: %v", err)
    }
    
    // Connect using GORM
    db, err := gorm.Open(postgres.Open(connStr), &gorm.Config{})
    if err != nil {
        cleanup()
        t.Fatalf("Failed to connect to database: %v", err)
    }
    
    // Run GORM AutoMigrate with internal models
    if err := db.AutoMigrate(&models.Product{} /* add other models */); err != nil {
        cleanup()
        t.Fatalf("Failed to run migrations: %v", err)
    }
    
    return db, cleanup
}

// Create helper function for truncation (reusable across all tests)
func truncateTables(db *gorm.DB, tables ...string) {
    // Truncate in reverse order (children before parents)
    for i := len(tables) - 1; i >= 0; i-- {
        db.Exec(fmt.Sprintf("TRUNCATE TABLE %s CASCADE", tables[i]))
    }
}

// Usage in tests
func TestProductCreate(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products", "categories")  // Children before parents
    
    // Test implementation...
}
```

**Benefits of Truncation Strategy:**
- Tests real production behavior with commits
- Works with any code structure
- Simple and fast (~1-5ms overhead)
- Handles foreign key constraints with CASCADE

**Test Database AI Agent Requirements:**
- Use `testcontainers.PostgresContainer` for automatic PostgreSQL lifecycle
- Setup schema with GORM `AutoMigrate` (matches production)
- Cleanup with `defer container.Terminate(ctx)` pattern
- Truncate tables after each test using `defer` pattern
- Use `TRUNCATE ... CASCADE` to handle foreign key constraints
- Truncate in reverse dependency order (children before parents)

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
                
                // Use cmpopts.IgnoreFields for generated fields (ID, timestamps)
                opts := cmp.Options{
                    protocmp.Transform(),
                    protocmp.IgnoreFields(&pb.Product{}, "id", "created_at", "updated_at"),
                }
                
                if diff := cmp.Diff(tc.wantProduct, &actual, opts...); diff != "" {
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
    
    // Build expected from known input values only
    expected := &pb.Product{
        Name:        "Test Product",
        Sku:         "SKU-001",
        Price:       1999,
        Description: "A test product",
    }
    
    // Use cmpopts.IgnoreFields for generated fields (ID, timestamps)
    opts := cmp.Options{
        protocmp.Transform(),
        protocmp.IgnoreFields(&pb.Product{}, "id", "created_at", "updated_at"),
    }
    
    if diff := cmp.Diff(expected, actual, opts...); diff != "" {
        t.Errorf("Product mismatch (-want +got):\n%s", diff)
    }
}
```

### Handling Dynamic Fields (IDs, Timestamps)

For fields that are generated (UUIDs, timestamps), use `cmpopts.IgnoreFields` or `protocmp.IgnoreFields`:

```go
// Build expected from known input values only - do NOT copy from actual
expected := &pb.Product{
    Name: "Test Product",   // ✅ From request - use expected value
    Sku:  "SKU-001",        // ✅ From request - use expected value
}

// Use IgnoreFields for generated fields
opts := cmp.Options{
    protocmp.Transform(),
    protocmp.IgnoreFields(&pb.Product{}, "id", "created_at", "updated_at"),
}

if diff := cmp.Diff(expected, actual, opts...); diff != "" {
    t.Errorf("Product mismatch (-want +got):\n%s", diff)
}
```

### Comparing Lists/Slices

```go
// ✅ CORRECT: Compare entire list with IgnoreFields for generated fields
expected := &pb.ListProductsResponse{
    Products: []*pb.Product{
        {Name: "Product A", Sku: "SKU-A"},
        {Name: "Product B", Sku: "SKU-B"},
    },
    Total: 2,
}

opts := cmp.Options{
    protocmp.Transform(),
    protocmp.IgnoreFields(&pb.Product{}, "id", "created_at", "updated_at"),
}

if diff := cmp.Diff(expected, actual, opts...); diff != "" {
    t.Errorf("List mismatch (-want +got):\n%s", diff)
}

// ❌ WRONG: Only checking length
if len(actual.Products) != 2 {
    t.Error("wrong count")
}
```

### Ignoring Specific Fields

For protobuf types, use `protocmp.IgnoreFields` with proto field names (snake_case):

```go
// For protobuf types - use protocmp.IgnoreFields with proto field names
opts := cmp.Options{
    protocmp.Transform(),
    protocmp.IgnoreFields(&pb.Product{}, "id", "created_at", "updated_at"),
}

if diff := cmp.Diff(expected, actual, opts...); diff != "" {
    t.Errorf("Mismatch (-want +got):\n%s", diff)
}
```

For non-protobuf Go structs, use `cmpopts.IgnoreFields` with Go field names (PascalCase):

```go
// For regular Go structs - use cmpopts.IgnoreFields with Go field names
opts := cmp.Options{
    cmpopts.IgnoreFields(pim.Product{}, "Id", "CreatedAt", "UpdatedAt"),
    cmpopts.EquateEmpty(),
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
// ✅ ALWAYS use cmp.Diff with IgnoreFields for generated fields
// For protobuf types:
opts := cmp.Options{
    protocmp.Transform(),
    protocmp.IgnoreFields(&pb.Product{}, "id", "created_at", "updated_at"),
}
if diff := cmp.Diff(expected, actual, opts...); diff != "" {
    t.Errorf("Mismatch (-want +got):\n%s", diff)
}

// For regular Go structs:
opts := cmp.Options{
    cmpopts.IgnoreFields(Product{}, "Id", "CreatedAt", "UpdatedAt"),
    cmpopts.EquateEmpty(),
}
if diff := cmp.Diff(expected, actual, opts...); diff != "" {
    t.Errorf("Mismatch (-want +got):\n%s", diff)
}
```

## Root Cause Tracing (Debugging Discipline)

When tests fail or bugs are reported, apply systematic root cause analysis before implementing fixes.

### Core Principles

- Problems MUST be traced backward through the call chain to find the original trigger
- Symptoms MUST be distinguished from root causes
- Fixes MUST address the source of the problem, NOT work around symptoms
- Test cases MUST NOT be removed or weakened to make tests pass
- Debuggers and logging MUST be used to understand control flow
- Multiple potential causes MUST be systematically eliminated
- Root cause MUST be verified through testing before closing the issue

### No-Give-Up Rule (NON-NEGOTIABLE)

AI agents MUST NEVER abandon a problem by:
- Reverting to "simpler" approaches that avoid the actual issue
- Saying "this won't work easily with this architecture"
- Removing tests or features because they're "too complex"
- Giving up after initial failures without exhausting root cause analysis

Instead, AI agents MUST:
1. Continue investigating until the root cause is found
2. Try multiple hypotheses systematically
3. Read source code to understand the actual implementation
4. Document findings even if the fix requires architectural changes
5. Only escalate to the user when genuinely blocked after thorough investigation

### Debugging Process

#### Step 1: Reproduce
Create a reliable reproduction case:
- Write a failing integration test
- Use table-driven design with descriptive name
- Test through root mux ServeHTTP

```go
{name: "BUG-123: PUT product returns 500 for valid request"}
```

#### Step 2: Observe
Gather evidence through logs, debugger, tests:
- Add temporary logging in service layers
- Check database state before/after operations
- Examine error responses and stack traces
- Use `t.Logf()` to output intermediate values

```go
t.Logf("Request body: %s", string(reqBody))
t.Logf("Response: status=%d body=%s", rec.Code, rec.Body.String())
t.Logf("DB state: %+v", product)
```

#### Step 3: Hypothesize
Form theories about root cause:
- List all possible causes
- Rank by likelihood based on evidence
- Consider recent changes that might have introduced the issue

Common Go backend root causes:
- **Validation**: Missing or incorrect validation in service layer
- **Database**: Wrong GORM query, missing preload, constraint violation
- **Serialization**: JSON field tags, ogen type marshaling
- **Context**: Missing `WithContext(ctx)`, context cancellation
- **Concurrency**: Race condition, missing mutex

#### Step 4: Test
Design experiments to validate/invalidate hypotheses:
- Test one hypothesis at a time
- Use binary search to narrow down the problem
- Add temporary debug code to verify assumptions

```go
var count int64
db.Model(&models.Product{}).Count(&count)
t.Logf("Product count after create: %d", count)
```

#### Step 5: Fix
Implement fix addressing root cause:
- Fix the source, not the symptom
- Ensure fix doesn't break existing functionality
- Keep fix minimal and focused

#### Step 6: Verify
Run the reproduction test:
```bash
go test -v -run "BUG-" ./...
```

Run full test suite:
```bash
go test -v -race ./...
```

#### Step 7: Document
Update docs/tests to prevent regression:
- Keep the regression test with bug reference in name
- Update API documentation if behavior was unclear
- Add comments explaining non-obvious fixes

### Common Error Patterns

| Error | Possible Causes |
|-------|-----------------|
| "record not found" but data exists | Wrong ID field, soft delete filtering, incorrect query |
| "duplicate key" on create | Unique constraint, ID not auto-generated |
| "context canceled" unexpectedly | Handler not using `r.Context()`, client disconnect |
| "invalid request" but looks valid | JSON field case mismatch, missing Content-Type |
| Test passes locally, fails in CI | Race condition, time-dependent, order-dependent |

### Test Fix Discipline

When tests fail:
- NEVER use `t.Skip()` to avoid fixing a complicated test
- ALWAYS trace the root cause before implementing fixes
- When API changes break tests, update assertions to match new behavior
- Remove tests only for removed functionality

## Test Package Organization

Integration tests SHOULD be in a separate `tests/` package:

```
tests/
├── testutil_test.go     # setupTestDB, truncateTables, fixture creators
└── product_test.go      # Integration tests
```

**Benefits**:
- Keeps service packages clean
- Allows tests to import from multiple packages without circular dependencies
- Test helpers are co-located with tests

## AI Agent Requirements

- MUST use real PostgreSQL via testcontainers (NO mocking)
- MUST test through root mux ServeHTTP (NOT individual handlers)
- MUST use table-driven tests with descriptive names
- **MUST use ogen API structs for request bodies (NEVER plain JSON strings)**
- MUST use `cmp.Diff()` for ALL struct comparisons (NEVER individual field assertions)
- MUST fill ALL struct fields completely (no empty arrays, no nil nested structs)
- MUST have at least 2 elements in every array/slice field
- MUST populate every nested struct recursively to primitive values
- MUST use realistic production-like values (not "test", "foo", "bar")
- **MUST NEVER copy values from execution results into expected structs** - use `cmpopts.IgnoreFields()` for generated fields (IDs, timestamps)
- MUST build expected structs entirely from input data
- MUST verify tests FAIL before implementation
- MUST run tests after EVERY code change
- MUST apply root cause tracing for test failures - no superficial fixes
- MUST write failing reproduction test BEFORE attempting any fix
- MUST document root cause analysis process

## See Also

- `/theplant.bugfix` - Bug fix workflow with reproduction-first debugging
- `/theplant.system-exploration` - Trace code paths before writing tests
- `/theplant.errors` - Error handling strategy
- `/theplant.openapi` - OpenAPI code generation with ogen
