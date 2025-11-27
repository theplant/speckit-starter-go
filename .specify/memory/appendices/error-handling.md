# Error Handling Appendix

**Part of**: [Go Project Constitution](../constitution.md)  
**Covers**: Principle XIII

This appendix provides detailed implementation guidelines for comprehensive error handling using a two-layer strategy with sentinel errors and HTTP error code singletons.

---

## Error Handling Strategy

### XIII. Comprehensive Error Handling

Error handling MUST use a two-layer strategy: sentinel errors for internal flow and singleton HTTP error codes for client responses:

**Service Layer (Internal Error Flow)**:
- Errors MUST be wrapped using `fmt.Errorf()` with the `%w` verb to preserve error chains
- Sentinel errors (package-level variables) MUST be defined for domain-specific error types
- Error messages MUST add contextual information at each layer (operation name, parameters, entity IDs)
- Errors MUST NEVER be swallowed without logging or returning them upward
- Service layer MUST wrap errors with business context before returning to handlers

**HTTP Layer (Client-Facing Responses)**:
- HTTP error codes and messages MUST be defined using singleton struct instances
- Error codes MUST NOT be hardcoded strings scattered throughout the codebase
- Error type checking MUST use `errors.Is()` and `errors.As()` (NOT string comparison)
- HTTP handlers MUST translate service errors to appropriate error codes
- HTTP responses MUST NOT expose internal error details or stack traces to clients
- The singleton MUST use struct literal initialization for compile-time safety
- Error definitions MUST be organized by category (validation, not found, conflict, internal)

**Testing**:
- Tests MUST verify error wrapping chains using `errors.Is()` and `errors.As()`
- Tests MUST verify HTTP handlers return correct error codes for each error type
- **ALL defined sentinel errors MUST have corresponding test cases** (each ErrXxx must be tested)
- **ALL HTTP error codes MUST have corresponding test cases** (each Errors.Xxx must be tested)
- Tests MUST cover both success and error paths for every operation
- Error test cases MUST verify the complete error flow: Service (sentinel) → Handler (HTTP code) → Client (response)

**Rationale**: This two-layer approach provides type-safe error handling at both service and HTTP layers. Sentinel errors enable type-safe checking with `errors.Is()` across wrapped errors, while the HTTP singleton provides consistent client-facing error codes. Error wrapping creates an "error breadcrumb trail" for debugging without stack traces (Go standard library doesn't provide stack traces). The singleton pattern prevents hardcoded strings and provides IDE autocomplete. Together, these create a robust error handling strategy that's type-safe, debuggable, and maintains clean separation between internal and external concerns.

**CRITICAL - Error Testing**: Every defined error (both sentinel errors in services and HTTP error codes in handlers) MUST have at least one test case that triggers and verifies it. Untested error paths are bugs waiting to happen in production. Tests MUST verify the complete error flow from service (sentinel error) through handler (HTTP code mapping) to client (HTTP response).

**Complete Error Handling Example**:

**Step 1: Define Sentinel Errors (Service Layer)**:
```go
// services/errors.go
package services

import "errors"

// Domain-specific sentinel errors for type-safe error checking
var (
    ErrProductNotFound  = errors.New("product not found")
    ErrDuplicateSKU     = errors.New("SKU already exists")
    ErrInvalidSKU       = errors.New("invalid SKU format")
    ErrMissingRequired  = errors.New("required field missing")
    ErrValueOutOfRange  = errors.New("value out of range")
)
```

**Step 2: Define HTTP Error Codes with Service Error Mapping (Handler Layer)**:
```go
// handlers/error_codes.go
package handlers

import (
    "net/http"
    "github.com/yourapp/services"
)

// ErrorCode for HTTP responses with optional service error mapping
type ErrorCode struct {
    Code       string
    Message    string
    HTTPStatus int
    ServiceErr error  // Optional: Maps to service sentinel error for automatic checking
}

// Singleton instance with all HTTP error definitions
var Errors = struct {
    // Validation errors (400)
    InvalidRequest   ErrorCode
    ValidationFailed ErrorCode
    MissingRequired  ErrorCode
    ValueOutOfRange  ErrorCode
    InvalidType      ErrorCode
    
    // Not found errors (404)
    ProductNotFound  ErrorCode
    TemplateNotFound ErrorCode
    
    // Conflict errors (409)
    DuplicateSKU     ErrorCode
    AlreadyExists    ErrorCode
    
    // Internal errors (500)
    InternalError    ErrorCode
}{
    // Validation errors - mapped to service errors
    InvalidRequest:   ErrorCode{"INVALID_REQUEST", "Invalid request body", http.StatusBadRequest, services.ErrInvalidRequest},
    MissingRequired:  ErrorCode{"MISSING_REQUIRED", "Required field missing", http.StatusBadRequest, services.ErrMissingRequired},
    ValueOutOfRange:  ErrorCode{"VALUE_OUT_OF_RANGE", "Value out of range", http.StatusBadRequest, services.ErrValueOutOfRange},
    InvalidType:      ErrorCode{"INVALID_TYPE", "Invalid type", http.StatusBadRequest, services.ErrInvalidType},
    
    // Not found errors - mapped to service errors
    ProductNotFound:  ErrorCode{"PRODUCT_NOT_FOUND", "Product not found", http.StatusNotFound, services.ErrProductNotFound},
    TemplateNotFound: ErrorCode{"TEMPLATE_NOT_FOUND", "Template not found", http.StatusNotFound, services.ErrTemplateNotFound},
    
    // Conflict errors - mapped to service errors
    DuplicateSKU:     ErrorCode{"DUPLICATE_SKU", "SKU already exists", http.StatusConflict, services.ErrDuplicateSKU},
    AlreadyExists:    ErrorCode{"ALREADY_EXISTS", "Resource already exists", http.StatusConflict, services.ErrAlreadyExists},
    
    // Internal errors - no mapping (catch-all)
    InternalError:    ErrorCode{"INTERNAL_ERROR", "Internal server error", http.StatusInternalServerError, nil},
}

// AllErrors returns a slice of all error codes for iteration
func AllErrors() []ErrorCode {
    return []ErrorCode{
        Errors.InvalidRequest,
        Errors.MissingRequired,
        Errors.ValueOutOfRange,
        Errors.InvalidType,
        Errors.ProductNotFound,
        Errors.TemplateNotFound,
        Errors.DuplicateSKU,
        Errors.AlreadyExists,
        Errors.InternalError,
    }
}
```

**Step 3: Service Layer - Wrap with Sentinel Errors**:
```go
// services/product_service.go
package services

import (
    "context"
    "errors"
    "fmt"
    "gorm.io/gorm"
)

func (s *productService) Get(ctx context.Context, req *pb.GetProductRequest) (*pb.Product, error) {
    var product Product
    
    if err := s.db.WithContext(ctx).First(&product, "id = ?", req.Id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            // Wrap with sentinel error for type-safe checking
            return nil, fmt.Errorf("get product %s: %w", req.Id, ErrProductNotFound)
        }
        // Wrap other errors with context
        return nil, fmt.Errorf("query product %s: %w", req.Id, err)
    }
    
    return toProto(&product), nil
}

func (s *productService) Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    // Validation returns sentinel error
    if req.Name == "" {
        return nil, fmt.Errorf("product name: %w", ErrMissingRequired)
    }
    
    product := &Product{Name: req.Name, SKU: req.Sku}
    
    if err := s.db.WithContext(ctx).Create(product).Error; err != nil {
        if isDuplicateKeyError(err) {
            return nil, fmt.Errorf("create product SKU %s: %w", req.Sku, ErrDuplicateSKU)
        }
        return nil, fmt.Errorf("create product: %w", err)
    }
    
    return toProto(product), nil
}
```

**Step 4: Handler Layer - Simplified Error Mapping**:
```go
// handlers/product_handler.go
package handlers

import (
    "context"
    "errors"
)

func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    var req pb.CreateProductRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        RespondWithError(w, Errors.InvalidRequest)
        return
    }
    
    // Call service
    product, err := h.service.Create(ctx, &req)
    if err != nil {
        // Automatic error mapping - NO SWITCH NEEDED!
        HandleServiceError(w, err)
        return
    }
    
    json.NewEncoder(w).Encode(product)
}

// HandleServiceError automatically maps service errors to HTTP responses
func HandleServiceError(w http.ResponseWriter, err error) {
    // Check context errors first
    if errors.Is(err, context.Canceled) {
        http.Error(w, "Request cancelled", 499)
        return
    }
    if errors.Is(err, context.DeadlineExceeded) {
        http.Error(w, "Request timeout", 504)
        return
    }
    
    // Iterate through all error codes and check ServiceErr mapping
    for _, errCode := range AllErrors() {
        if errCode.ServiceErr != nil && errors.Is(err, errCode.ServiceErr) {
            RespondWithError(w, errCode)
            return
        }
    }
    
    // Default: Internal error (don't expose details)
    RespondWithError(w, Errors.InternalError)
}

// Helper function
func RespondWithError(w http.ResponseWriter, errCode ErrorCode) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(errCode.HTTPStatus)
    json.NewEncoder(w).Encode(map[string]string{
        "code":    errCode.Code,
        "message": errCode.Message,
    })
}
```

**Complete Error Flow (Simplified)**:

1. Service validates → Returns: fmt.Errorf("name: %w", ErrMissingRequired)
2. Handler receives error
3. HandleServiceError() iterates through AllErrors()
4. Finds: Errors.MissingRequired.ServiceErr == services.ErrMissingRequired ✅
5. Auto-maps: RespondWithError(w, Errors.MissingRequired)
6. Client receives: {"code": "MISSING_REQUIRED", "message": "...", "status": 400}

**Benefits of ServiceErr Field**:
- ✅ **No switch statement** - automatic error mapping via iteration
- ✅ **Single source of truth** - mapping defined once in ErrorCode definition
- ✅ **Easy to add errors** - just add to Errors struct with ServiceErr field
- ✅ **Type-safe** - still uses errors.Is() under the hood
- ✅ **Maintainable** - add new error by adding one line to Errors singleton
- ✅ **Flexible** - ServiceErr is optional (nil for HTTP-only errors)

**Error Testing Requirements**:

Every sentinel error and HTTP error code MUST have test cases following constitutional testing principles:

```go
// Example: Testing ErrDuplicateSKU (complete constitutional pattern)
func TestProductAPI_Create_DuplicateSKU(t *testing.T) {
    // Principle I: Integration test with real database
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    // Principle IV: Create database fixture (existing product with SKU)
    existingProduct := createTestProduct(db, map[string]interface{}{
        "name": "Existing Product",
        "sku":  "DUP-001",
    })
    
    // Setup service and routes
    service := NewProductService(db).Build()
    mux := SetupRoutes(service)
    
    // Principle VI: Use protobuf structs
    reqData := &pb.CreateProductRequest{
        Name: "New Product",
        Sku:  "DUP-001",  // Duplicate SKU
    }
    body, _ := json.Marshal(reqData)
    req := httptest.NewRequest("POST", "/api/products", bytes.NewReader(body))
    rec := httptest.NewRecorder()
    
    // Principle V: Test through root mux ServeHTTP (NOT handler method)
    mux.ServeHTTP(rec, req)
    
    // Verify complete error flow:
    // 1. HTTP status code
    if rec.Code != http.StatusConflict {
        t.Errorf("Expected status %d, got %d", http.StatusConflict, rec.Code)
    }
    
    // 2. Error response structure
    var errResp struct {
        Code    string `json:"code"`
        Message string `json:"message"`
    }
    if err := json.NewDecoder(rec.Body).Decode(&errResp); err != nil {
        t.Fatalf("Failed to decode error response: %v", err)
    }
    
    // 3. Verify error code matches expected
    if errResp.Code != "DUPLICATE_SKU" {
        t.Errorf("Expected error code DUPLICATE_SKU, got %s", errResp.Code)
    }
    
    // 4. Verify original product unchanged in database
    var count int64
    db.Model(&Product{}).Where("sku = ?", "DUP-001").Count(&count)
    if count != 1 {
        t.Errorf("Expected 1 product with SKU DUP-001, got %d", count)
    }
}

// Example: Testing multiple error scenarios (table-driven)
func TestProductAPI_Create_ValidationErrors(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    service := NewProductService(db).Build()
    mux := SetupRoutes(service)
    
    // Principle II: Table-driven test design
    testCases := []struct {
        name           string
        request        *pb.CreateProductRequest
        expectedStatus int
        expectedCode   string
    }{
        {
            name:           "Missing required name",
            request:        &pb.CreateProductRequest{Sku: "TEST-001"},
            expectedStatus: http.StatusBadRequest,
            expectedCode:   "MISSING_REQUIRED",
        },
        {
            name:           "Empty name",
            request:        &pb.CreateProductRequest{Name: "", Sku: "TEST-002"},
            expectedStatus: http.StatusBadRequest,
            expectedCode:   "MISSING_REQUIRED",
        },
        {
            name:           "Invalid SKU format",
            request:        &pb.CreateProductRequest{Name: "Product", Sku: "invalid@sku"},
            expectedStatus: http.StatusBadRequest,
            expectedCode:   "INVALID_SKU",
        },
    }
    
    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            body, _ := json.Marshal(tc.request)
            req := httptest.NewRequest("POST", "/api/products", bytes.NewReader(body))
            rec := httptest.NewRecorder()
            
            // Test through root mux
            mux.ServeHTTP(rec, req)
            
            if rec.Code != tc.expectedStatus {
                t.Errorf("Expected status %d, got %d", tc.expectedStatus, rec.Code)
            }
            
            var errResp struct {
                Code    string `json:"code"`
                Message string `json:"message"`
            }
            json.NewDecoder(rec.Body).Decode(&errResp)
            
            if errResp.Code != tc.expectedCode {
                t.Errorf("Expected error code %s, got %s", tc.expectedCode, errResp.Code)
            }
        })
    }
}
```

**Key Testing Requirements**:
- ✅ Test through root mux ServeHTTP (Principle V - tests routing)
- ✅ Use builder pattern for service construction
- ✅ Use protobuf structs for requests (Principle VI)
- ✅ Create real database fixtures (Principle IV)
- ✅ Table-driven design for multiple error scenarios (Principle II)
- ✅ Verify complete error flow: Service → Handler → HTTP Response
- ✅ Test both error code and HTTP status code
- ✅ Verify database state remains consistent

**Why Two Layers (Simplified)**:
- **Service sentinel errors**: Domain logic errors (ErrProductNotFound) - ALL must be tested
- **HTTP error codes**: Client-facing codes with automatic mapping via ServiceErr field - ALL must be tested
- **No manual switch**: AllErrors() + ServiceErr field handles mapping
- **Clean handlers**: Just call HandleServiceError(w, err) - done!


### Code Review Requirements

All pull requests MUST be reviewed against these constitutional requirements:


**Principle XIII: Comprehensive Error Handling**
- Reviewers MUST verify sentinel errors defined for domain-specific errors
- Reviewers MUST verify errors are wrapped with `fmt.Errorf("%w", err)` (NOT `%v`)
- Reviewers MUST verify error messages add contextual information at each layer
- Reviewers MUST verify error checking uses `errors.Is()` and `errors.As()` (NOT string comparison)
- Reviewers MUST verify errors are never swallowed without logging or returning
- Reviewers MUST verify HTTP error codes use singleton struct instances (no hardcoded strings)
- Reviewers MUST verify ErrorCode.ServiceErr field maps to service sentinel errors
- Reviewers MUST verify HandleServiceError uses AllErrors() iteration (no switch statement)
- Reviewers MUST verify HTTP handlers do NOT expose internal error details to clients
- Reviewers MUST verify tests use `errors.Is()` and `errors.As()` for error validation
- **Reviewers MUST verify ALL sentinel errors have test cases** (every ErrXxx tested)
- **Reviewers MUST verify ALL HTTP error codes have test cases** (every Errors.Xxx tested)
- **Reviewers MUST verify both success and error paths tested for every operation**
