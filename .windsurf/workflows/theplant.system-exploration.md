---
description: Apply System Exploration Protocol before writing tests - trace BOTH read paths (storage → UI) AND write paths (UI → storage) to understand complete data flow.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Before writing tests for an unfamiliar codebase, trace **BOTH** the read path (DB → API response) **AND** the write path (API request → DB) to understand complete data flow.

## Rationale

Root cause analysis requires understanding the full system. This protocol ensures AI agents understand the **complete bidirectional data flow** before writing tests or debugging issues. This prevents superficial fixes and ensures tests validate actual behavior.

## Core Principles

For every feature, trace **BOTH** code paths:

**READ Path (GET requests):**
```
HTTP GET → Handler → Service → Repository/GORM → Database
```

**WRITE Path (POST/PUT/DELETE requests):**
```
HTTP POST/PUT/DELETE → Handler → Service (validation) → Repository/GORM → Database
```

The READ trace reveals:
- What data entities the endpoint returns
- What fields are included in responses
- What relationships are loaded (preloads, joins)
- What query filters are applied

The WRITE trace reveals:
- What validation is applied in the service layer
- What sentinel errors can be returned
- What database constraints exist
- What transactions are used

## Execution Steps

First, create a task plan using `update_plan`:

```
Call update_plan with:
- explanation: "System exploration for <feature/endpoint>"
- plan: [
    {"step": "Read route definitions in handlers/routes.go", "status": "pending"},
    {"step": "Trace READ path (DB → API response)", "status": "pending"},
    {"step": "Trace WRITE path (API request → DB)", "status": "pending"},
    {"step": "Document code trace in test file", "status": "pending"},
    {"step": "Identify required test data and fixtures", "status": "pending"},
    {"step": "Identify test scenarios (success + error cases)", "status": "pending"},
    {"step": "Write tests covering all scenarios", "status": "pending"}
  ]
```

Then execute each step, updating status to `in_progress` before starting and `completed` after finishing.

### Step 1: Read Route Definitions

Explore `handlers/routes.go` to find route registration:

```go
// Look for patterns like:
mux.HandleFunc("GET /api/v1/products/{id}", h.Get)
mux.HandleFunc("POST /api/v1/products", h.Create)
mux.HandleFunc("PUT /api/v1/products/{id}", h.Update)
mux.HandleFunc("DELETE /api/v1/products/{id}", h.Delete)
```

Note:
- HTTP methods (GET, POST, PUT, DELETE)
- Path patterns and parameters (`{id}`)
- Handler method names

### Step 2: Trace READ Path (NON-NEGOTIABLE)

For each GET endpoint, trace the full code path:

1. **Read the handler** to find which service method it calls
2. **Read the service method** to find database queries
3. **Read the GORM queries** to understand what data is loaded
4. **Check for preloads** to understand relationships

```go
// Handler (handlers/product_handler.go)
func (h *ProductHandler) Get(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    product, err := h.service.Get(ctx, &pb.GetProductRequest{Id: id})
    // ...
}

// Service (services/product_service.go)
func (s *productService) Get(ctx context.Context, req *pb.GetProductRequest) (*pb.Product, error) {
    var product models.Product
    err := s.db.WithContext(ctx).
        Preload("Category").
        Preload("Variants").
        First(&product, "id = ?", req.Id).Error
    // ...
}
```

### Step 3: Trace WRITE Path (NON-NEGOTIABLE)

For each POST/PUT/DELETE endpoint, trace the full code path:

1. **Read the handler** to see request decoding
2. **Read the service method** to find validation logic
3. **Identify sentinel errors** that can be returned
4. **Check database operations** (Create, Update, Delete)
5. **Check for transactions** if multiple operations

```go
// Handler (thin wrapper)
func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req pb.CreateProductRequest
    DecodeProtoJSON(r, &req)
    product, err := h.service.Create(ctx, &req)
    // ...
}

// Service (validation + business logic)
func (s *productService) Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    // Validation
    if req.Name == "" {
        return nil, fmt.Errorf("product name: %w", ErrMissingRequired)
    }
    
    // Database operation
    product := &models.Product{Name: req.Name, SKU: req.Sku}
    if err := s.db.WithContext(ctx).Create(product).Error; err != nil {
        if isDuplicateKeyError(err) {
            return nil, fmt.Errorf("SKU %s: %w", req.Sku, ErrDuplicateSKU)
        }
        return nil, fmt.Errorf("create product: %w", err)
    }
    // ...
}
```

### Step 4: Document the Code Trace

Document BOTH traces as comments in the test file:

```go
// ─────────────────────────────────────────────────────────────
// Products API (System Exploration Protocol)
// 
// READ Path (GET /api/v1/products/{id}):
//   handlers/product_handler.go:Get()
//     → services/product_service.go:Get()
//       → GORM query with Preload("Category", "Variants")
//         → models.Product with relationships
//
// WRITE Path (POST /api/v1/products):
//   handlers/product_handler.go:Create()
//     → DecodeProtoJSON into pb.CreateProductRequest
//       → services/product_service.go:Create()
//         → Validates: Name required, SKU format
//         → Sentinel errors: ErrMissingRequired, ErrDuplicateSKU
//         → GORM Create with unique constraint on SKU
//
// WRITE Path (PUT /api/v1/products/{id}):
//   handlers/product_handler.go:Update()
//     → services/product_service.go:Update()
//       → Validates: ID required, Name required
//       → Sentinel errors: ErrProductNotFound, ErrMissingRequired
//       → GORM Updates (selective)
//
// WRITE Path (DELETE /api/v1/products/{id}):
//   handlers/product_handler.go:Delete()
//     → services/product_service.go:Delete()
//       → Sentinel errors: ErrProductNotFound
//       → GORM Delete (soft delete if DeletedAt field exists)
// ─────────────────────────────────────────────────────────────
```

### Step 5: Identify Required Test Data

From the trace, identify:

| Question | How to Find |
|----------|-------------|
| What entities? | Look at GORM models in `internal/models/` |
| What fields required? | Check service validation logic |
| What relationships? | Look for `Preload()` calls and foreign keys |
| What constraints? | Check database migrations and model tags |

### Step 6: Identify Test Scenarios

**READ Tests:**
| Scenario | Setup | Expected |
|----------|-------|----------|
| Found | Insert fixture | 200 with data |
| Not found | No fixture | 404 |
| With relationships | Insert related fixtures | Response includes related data |

**WRITE Tests:**
| Scenario | Request | Expected |
|----------|---------|----------|
| Valid create | All required fields | 201 with created entity |
| Missing required | Omit required field | 400 with ErrMissingRequired |
| Duplicate | Insert existing, create duplicate | 409 with ErrDuplicateSKU |
| Update existing | Valid update request | 200 with updated entity |
| Update not found | Non-existent ID | 404 with ErrProductNotFound |
| Delete existing | Valid ID | 204 |
| Delete not found | Non-existent ID | 404 |

### Step 7: Write Tests

```go
func TestProductAPI(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products", "categories")
    
    service := services.NewProductService(db).Build()
    mux := handlers.NewRouter(service, db).Build()
    
    testCases := []struct {
        name           string
        method         string
        path           string
        body           proto.Message
        setupFixtures  func()
        wantStatus     int
        wantErr        *handlers.ErrorCode
    }{
        // ─────────────────────────────────────────────────────────────
        // READ Tests
        // ─────────────────────────────────────────────────────────────
        {
            name:   "GET product - found",
            method: "GET",
            path:   "/api/v1/products/prod-123",
            setupFixtures: func() {
                db.Create(&models.Product{ID: "prod-123", Name: "Test"})
            },
            wantStatus: http.StatusOK,
        },
        {
            name:       "GET product - not found",
            method:     "GET",
            path:       "/api/v1/products/nonexistent",
            wantStatus: http.StatusNotFound,
            wantErr:    &handlers.Errors.ProductNotFound,
        },
        
        // ─────────────────────────────────────────────────────────────
        // WRITE Tests (Create)
        // ─────────────────────────────────────────────────────────────
        {
            name:   "POST product - valid",
            method: "POST",
            path:   "/api/v1/products",
            body:   &pb.CreateProductRequest{Name: "New Product", Sku: "SKU-001"},
            wantStatus: http.StatusCreated,
        },
        {
            name:       "POST product - missing name",
            method:     "POST",
            path:       "/api/v1/products",
            body:       &pb.CreateProductRequest{Sku: "SKU-001"},
            wantStatus: http.StatusBadRequest,
            wantErr:    &handlers.Errors.MissingRequired,
        },
        {
            name:   "POST product - duplicate SKU",
            method: "POST",
            path:   "/api/v1/products",
            body:   &pb.CreateProductRequest{Name: "Another", Sku: "DUP-SKU"},
            setupFixtures: func() {
                db.Create(&models.Product{Name: "Existing", SKU: "DUP-SKU"})
            },
            wantStatus: http.StatusConflict,
            wantErr:    &handlers.Errors.DuplicateSKU,
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
            
            mux.ServeHTTP(rec, req)
            
            if rec.Code != tc.wantStatus {
                t.Errorf("status = %d, want %d", rec.Code, tc.wantStatus)
            }
            
            if tc.wantErr != nil {
                var errResp pb.ErrorResponse
                protojson.Unmarshal(rec.Body.Bytes(), &errResp)
                if errResp.Code != tc.wantErr.Code {
                    t.Errorf("error code = %s, want %s", errResp.Code, tc.wantErr.Code)
                }
            }
        })
    }
}
```

## CRITICAL: AI agents MUST NOT

- Guess what data exists in the system
- Write tests that pass regardless of data
- Skip the service layer trace (validation, sentinel errors)
- Implement fixes without understanding the full code path
- Trace only READ paths and ignore WRITE paths
- Write only happy-path tests without error cases

## AI Agent Requirements

- MUST complete System Exploration Protocol before writing tests for unfamiliar code
- MUST trace **BOTH** READ and WRITE paths for every feature
- MUST document the code trace in test files
- MUST identify sentinel errors from service layer
- MUST apply Root Cause Tracing (Principle ROOT_CAUSE) when tests fail
- MUST verify data flow matches API spec (Principle SCHEMA_FIRST)
- MUST write tests for CRUD operations, not just reads
- MUST test error cases (validation failures, not found, conflicts)

## See Also

- `/theplant.integration-test` - Integration testing workflow
- `/theplant.bugfix` - Bug fix with reproduction-first debugging
- `/theplant.service` - Service layer architecture
- `/theplant.openapi` - OpenAPI code generation with ogen and error handling strategy
- `/theplant.root-cause-tracing` - Root cause analysis for debugging
