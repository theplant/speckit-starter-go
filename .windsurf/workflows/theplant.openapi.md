---
description: Generate Go types and server interfaces from OpenAPI specifications using ogen, with comprehensive error handling and services implementing the Handler interface.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Generate Go types and server interfaces from OpenAPI specifications using ogen (github.com/ogen-go/ogen). **Services implement the ogen Handler interface directly**, making them reusable as Go packages with OpenAPI-defined interfaces.

## Architecture

**CRITICAL**: The `services/` package contains an `OgenHandler` that implements the ogen-generated `Handler` interface and delegates to individual domain services. This allows:
- Clean separation between API contract (ogen Handler) and business logic (domain services)
- Services to be imported and used as Go packages by external applications
- OpenAPI spec defines the service contract (interface)
- Centralized nil-service checks and error handling

```
api/
├── openapi/           # OpenAPI specifications (SINGLE SOURCE OF TRUTH)
│   └── <domain>.yaml
└── gen/<domain>/      # ogen generated code
    ├── gen.go         # go:generate directive
    └── oas_*.go       # generated types, server, client

services/              # PUBLIC - business logic and ogen Handler
├── ogen_handler.go   # Implements api/gen/<domain>.Handler, delegates to services
├── <domain>_service.go  # Domain-specific business logic
├── errors.go         # Sentinel errors
└── migrations.go     # AutoMigrate() for external apps

handlers/              # HTTP routing and middleware
├── routes.go         # Router builder with ogen server setup
├── error_handler.go  # ogen ErrorHandler implementation
├── error_codes.go    # ErrorCodeMapping using ogen-generated ErrorResponse
└── middleware.go     # HTTP middlewares

internal/
├── models/           # GORM models (internal only)
└── config/
```

## Execution Steps

First, create a task plan using `update_plan`:

```
Call update_plan with:
- explanation: "OpenAPI code generation workflow for <domain>"
- plan: [
    {"step": "Install ogen if not available", "status": "pending"},
    {"step": "Define OpenAPI contract in api/openapi/<domain>.yaml", "status": "pending"},
    {"step": "Add go:generate directive to api/gen/<domain>/gen.go", "status": "pending"},
    {"step": "Generate Go code with go generate", "status": "pending"},
    {"step": "Implement OgenHandler in services/ogen_handler.go", "status": "pending"},
    {"step": "Implement domain services in services/<domain>_service.go", "status": "pending"},
    {"step": "Setup routes with ogen server in handlers/routes.go", "status": "pending"},
    {"step": "Verify compilation and tests pass", "status": "pending"}
  ]
```

Then execute each step, updating status to `in_progress` before starting and `completed` after finishing.

### Prerequisites

Install ogen if not available:
// turbo
```bash
go install github.com/ogen-go/ogen/cmd/ogen@latest
```

### Workflow Steps

#### 1. Define Contract
Create or update OpenAPI files in `api/openapi/<domain>.yaml` (YAML preferred) or `.json`

#### 2. Add go:generate Directive
Add to a Go file in the package (e.g., `api/gen/<domain>/gen.go`):

```go
package <domain>

//go:generate go run github.com/ogen-go/ogen/cmd/ogen@latest --target . --clean ../../openapi/<domain>.yaml
```

#### 3. Generate Go Code
// turbo
```bash
go generate ./...
```

#### 4. Implement OgenHandler in Services Package

**CRITICAL**: Create an `OgenHandler` in the services package that implements the ogen `Handler` interface and delegates to individual domain services:

```go
// services/ogen_handler.go
package services

import (
    "context"
    
    api "yourapp/api/gen/product"
)

// OgenHandler implements the ogen-generated api.Handler interface
// It delegates to the underlying domain services
type OgenHandler struct {
    productService  ProductService
    categoryService CategoryService
    // Add other services as needed
}

// OgenHandlerBuilder builds an OgenHandler with optional services
type OgenHandlerBuilder struct {
    productService  ProductService
    categoryService CategoryService
}

// NewOgenHandler creates a new OgenHandler builder
func NewOgenHandler() *OgenHandlerBuilder {
    return &OgenHandlerBuilder{}
}

// WithProductService adds product service
func (b *OgenHandlerBuilder) WithProductService(svc ProductService) *OgenHandlerBuilder {
    b.productService = svc
    return b
}

// WithCategoryService adds category service
func (b *OgenHandlerBuilder) WithCategoryService(svc CategoryService) *OgenHandlerBuilder {
    b.categoryService = svc
    return b
}

// Build creates the OgenHandler instance
func (b *OgenHandlerBuilder) Build() *OgenHandler {
    return &OgenHandler{
        productService:  b.productService,
        categoryService: b.categoryService,
    }
}

// Ensure OgenHandler implements api.Handler
var _ api.Handler = (*OgenHandler)(nil)

// ============================================================================
// Product Operations - delegate to ProductService
// ============================================================================

// CreateProduct implements api.Handler
func (h *OgenHandler) CreateProduct(ctx context.Context, req *api.CreateProductReq) (*api.Product, error) {
    if h.productService == nil {
        return nil, ErrMissingRequired
    }
    return h.productService.Create(ctx, req)
}

// GetProduct implements api.Handler
func (h *OgenHandler) GetProduct(ctx context.Context, params api.GetProductParams) (*api.Product, error) {
    if h.productService == nil {
        return nil, ErrMissingRequired
    }
    return h.productService.Get(ctx, params.ProductId)
}

// ListProducts implements api.Handler
func (h *OgenHandler) ListProducts(ctx context.Context, params api.ListProductsParams) (*api.ListProductsResponse, error) {
    if h.productService == nil {
        return nil, ErrMissingRequired
    }
    return h.productService.List(ctx, params)
}
```

#### 5. Implement Domain Services

Domain services contain the actual business logic:

```go
// services/product_service.go
package services

import (
    "context"
    "fmt"
    
    api "yourapp/api/gen/product"
    "yourapp/internal/models"
    "gorm.io/gorm"
)

// ProductService interface for product operations
type ProductService interface {
    Create(ctx context.Context, req *api.CreateProductReq) (*api.Product, error)
    Get(ctx context.Context, id string) (*api.Product, error)
    List(ctx context.Context, params api.ListProductsParams) (*api.ListProductsResponse, error)
}

// productServiceImpl implements ProductService
type productServiceImpl struct {
    db     *gorm.DB
    logger Logger  // Optional
}

// Builder pattern for dependency injection
type productServiceBuilder struct {
    db     *gorm.DB
    logger Logger
}

func NewProductService(db *gorm.DB) *productServiceBuilder {
    return &productServiceBuilder{db: db}
}

func (b *productServiceBuilder) WithLogger(logger Logger) *productServiceBuilder {
    b.logger = logger
    return b
}

func (b *productServiceBuilder) Build() ProductService {
    return &productServiceImpl{db: b.db, logger: b.logger}
}

// Create implements ProductService
func (s *productServiceImpl) Create(ctx context.Context, req *api.CreateProductReq) (*api.Product, error) {
    product := &models.Product{Name: req.Name, SKU: req.Sku}
    
    if err := s.db.WithContext(ctx).Create(product).Error; err != nil {
        if isDuplicateKeyError(err) {
            return nil, fmt.Errorf("create product SKU %s: %w", req.Sku, ErrDuplicateSKU)
        }
        return nil, fmt.Errorf("create product: %w", err)
    }
    
    return &api.Product{ID: product.ID, Name: product.Name, Sku: product.SKU}, nil
}

// Get implements ProductService
func (s *productServiceImpl) Get(ctx context.Context, id string) (*api.Product, error) {
    var product models.Product
    if err := s.db.WithContext(ctx).First(&product, "id = ?", id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("get product %s: %w", id, ErrProductNotFound)
        }
        return nil, fmt.Errorf("query product: %w", err)
    }
    return &api.Product{ID: product.ID, Name: product.Name, Sku: product.SKU}, nil
}
```

#### 6. Setup Routes with ogen Server

**CRITICAL**: Use the router builder pattern in `handlers/` to create the ogen server with error handling and middlewares:

```go
// handlers/routes.go
package handlers

import (
    "net/http"
    
    api "yourapp/api/gen/product"
    "yourapp/services"
)

// Middleware is a function that wraps an http.Handler
type Middleware func(http.Handler) http.Handler

// routerBuilder configures HTTP routes with optional services using ogen server
type routerBuilder struct {
    productService  services.ProductService
    categoryService services.CategoryService
    middlewares     []Middleware
}

// NewRouter creates a router builder
func NewRouter(productService services.ProductService) *routerBuilder {
    return &routerBuilder{
        productService: productService,
    }
}

// WithCategoryService adds category service
func (b *routerBuilder) WithCategoryService(svc services.CategoryService) *routerBuilder {
    b.categoryService = svc
    return b
}

// WithMiddlewares adds middlewares to the stack (applied in order)
func (b *routerBuilder) WithMiddlewares(mws ...Middleware) *routerBuilder {
    b.middlewares = append(b.middlewares, mws...)
    return b
}

// Build creates the configured HTTP handler using ogen server
func (b *routerBuilder) Build() (http.Handler, error) {
    // Build the ogen handler from services package
    handler := services.NewOgenHandler().
        WithProductService(b.productService).
        WithCategoryService(b.categoryService).
        Build()

    // Create ogen server with error handler
    server, err := api.NewServer(handler,
        api.WithErrorHandler(OgenErrorHandler),
    )
    if err != nil {
        return nil, err
    }

    // Apply middlewares (in order provided)
    var h http.Handler = server
    for _, mw := range b.middlewares {
        h = mw(h)
    }

    return h, nil
}

// DefaultMiddlewares returns the standard middleware stack for production
func DefaultMiddlewares() []Middleware {
    return []Middleware{
        recoveryMiddleware,
        TracingMiddleware(),
        LoggingMiddleware(),
        requestIDMiddleware,
        CORSMiddleware(DefaultCORSConfig()),
    }
}
```

## Two-Layer Error Handling Strategy

### Service Layer: Sentinel Errors

Define sentinel errors in `services/errors.go`:

```go
package services

import "errors"

var (
    ErrProductNotFound = errors.New("product not found")
    ErrDuplicateSKU    = errors.New("SKU already exists")
    ErrMissingRequired = errors.New("required field missing")
)
```

### HTTP Layer: Error Codes Using Generated ErrorResponse

**CRITICAL**: Do NOT define a custom `ErrorCode` struct. Instead, use the ogen-generated `api.ErrorResponse` type directly for error code definitions. This avoids duplication and ensures consistency with the OpenAPI contract.

Create `handlers/error_codes.go`:

```go
package handlers

import (
    "context"
    "net/http"
    
    api "yourapp/api/gen/<domain>"
    "yourapp/services"
)

// ErrorCodeMapping maps service sentinel errors to HTTP status and ErrorResponse
type ErrorCodeMapping struct {
    Response   api.ErrorResponse  // Use ogen-generated type
    HTTPStatus int
    ServiceErr error  // Maps to service sentinel error
}

var Errors = struct {
    MissingRequired  ErrorCodeMapping
    ProductNotFound  ErrorCodeMapping
    DuplicateSKU     ErrorCodeMapping
    RequestCancelled ErrorCodeMapping
    RequestTimeout   ErrorCodeMapping
    InternalError    ErrorCodeMapping
}{
    MissingRequired:  ErrorCodeMapping{api.ErrorResponse{Code: "MISSING_REQUIRED", Message: "Required field missing"}, http.StatusBadRequest, services.ErrMissingRequired},
    ProductNotFound:  ErrorCodeMapping{api.ErrorResponse{Code: "PRODUCT_NOT_FOUND", Message: "Product not found"}, http.StatusNotFound, services.ErrProductNotFound},
    DuplicateSKU:     ErrorCodeMapping{api.ErrorResponse{Code: "DUPLICATE_SKU", Message: "SKU already exists"}, http.StatusConflict, services.ErrDuplicateSKU},
    RequestCancelled: ErrorCodeMapping{api.ErrorResponse{Code: "REQUEST_CANCELLED", Message: "Request cancelled"}, 499, context.Canceled},
    RequestTimeout:   ErrorCodeMapping{api.ErrorResponse{Code: "REQUEST_TIMEOUT", Message: "Request timeout"}, http.StatusGatewayTimeout, context.DeadlineExceeded},
    InternalError:    ErrorCodeMapping{api.ErrorResponse{Code: "INTERNAL_ERROR", Message: "Internal server error"}, http.StatusInternalServerError, nil},
}

// AllErrors returns all error codes for iteration
func AllErrors() []ErrorCodeMapping {
    return []ErrorCodeMapping{
        Errors.MissingRequired,
        Errors.ProductNotFound,
        Errors.DuplicateSKU,
        Errors.RequestCancelled,
        Errors.RequestTimeout,
        Errors.InternalError,
    }
}
```

### Service: Wrap Errors with Context

```go
func (s *productService) Get(ctx context.Context, id string) (*api.Product, error) {
    var product models.Product
    if err := s.db.WithContext(ctx).First(&product, "id = ?", id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("get product %s: %w", id, ErrProductNotFound)
        }
        return nil, fmt.Errorf("query product %s: %w", id, err)
    }
    return &api.Product{ID: product.ID, Name: product.Name, Sku: product.SKU}, nil
}
```

### ogen ErrorHandler Integration

**CRITICAL**: When using ogen-generated servers, implement the `ErrorHandler` interface to map service errors to proper HTTP responses. This prevents internal error details from leaking to users.

Create `handlers/error_handler.go`:

```go
package handlers

import (
    "context"
    "encoding/json"
    "errors"
    "net/http"

    api "yourapp/api/gen/<domain>"
    "github.com/ogen-go/ogen/ogenerrors"
)

// CRITICAL: ErrorResponse MUST be defined in OpenAPI schema and generated by ogen
// Use the ogen-generated api.ErrorResponse type - do NOT define a custom ErrorCode struct

// OgenErrorHandler implements ogenerrors.ErrorHandler for ogen servers
// It maps service sentinel errors to user-friendly HTTP responses
func OgenErrorHandler(ctx context.Context, w http.ResponseWriter, r *http.Request, err error) {
    errMapping := mapServiceError(err)
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(errMapping.HTTPStatus)
    
    // Use the pre-defined ErrorResponse from the mapping
    resp := errMapping.Response
    
    // Include details in development only
    if !defaultErrorConfig.hideErrorDetails && err != nil {
        resp.Details = append(resp.Details, err.Error())
    }
    
    json.NewEncoder(w).Encode(resp)
}

func mapServiceError(err error) ErrorCodeMapping {
    // Check context errors first
    if errors.Is(err, context.Canceled) {
        return Errors.RequestCancelled
    }
    if errors.Is(err, context.DeadlineExceeded) {
        return Errors.RequestTimeout
    }
    
    // Check service sentinel errors
    for _, errMapping := range AllErrors() {
        if errMapping.ServiceErr != nil && errors.Is(err, errMapping.ServiceErr) {
            return errMapping
        }
    }
    
    // Check for ogen decode errors (validation failures)
    var decodeErr *ogenerrors.DecodeBodyError
    if errors.As(err, &decodeErr) {
        return Errors.MissingRequired
    }
    
    return Errors.InternalError
}
```

### Environment-Aware Error Details

```go
// Configuration
type errorResponseConfig struct {
    hideErrorDetails bool
}

var defaultErrorConfig = &errorResponseConfig{hideErrorDetails: false}

func SetHideErrorDetails(hide bool) {
    defaultErrorConfig.hideErrorDetails = hide
}
```

### Startup Configuration

```go
// cmd/api/main.go
func main() {
    if os.Getenv("HIDE_ERROR_DETAILS") == "true" {
        handlers.SetHideErrorDetails(true)
    }
    // ... rest of setup
}
```

### Example Error Responses

Development (default):
```json
{
    "code": "PRODUCT_NOT_FOUND",
    "message": "Product not found",
    "details": ["get product abc-123: product not found"]
}
```

Production (`HIDE_ERROR_DETAILS=true`):
```json
{
    "code": "PRODUCT_NOT_FOUND",
    "message": "Product not found"
}
```

### Testing Errors

```go
func TestProductAPI_Create_DuplicateSKU(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    
    // Create existing product
    createTestProduct(db, "existing", "DUP-001")
    
    productService := services.NewProductService(db).Build()
    handler, _ := handlers.NewRouter(productService).Build()
    
    // Try to create duplicate
    reqBody := api.CreateProductReq{Name: "New", Sku: "DUP-001"}
    body, _ := json.Marshal(reqBody)
    req := httptest.NewRequest("POST", "/api/v1/products", bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    rec := httptest.NewRecorder()
    
    handler.ServeHTTP(rec, req)
    
    // Verify error response
    if rec.Code != http.StatusConflict {
        t.Errorf("Expected 409, got %d", rec.Code)
    }
    
    var errResp api.ErrorResponse
    json.Unmarshal(rec.Body.Bytes(), &errResp)
    
    // Use error code definitions (NOT literal strings)
    if errResp.Code != Errors.DuplicateSKU.Response.Code {
        t.Errorf("Expected %s, got %s", Errors.DuplicateSKU.Response.Code, errResp.Code)
    }
}
```

Use in `cmd/api/main.go`:

```go
package main

import (
    "log"
    "net/http"
    "os"
    
    "yourapp/handlers"
    "yourapp/services"
)

func main() {
    db := setupDatabase()
    
    // Configure error details visibility
    if os.Getenv("HIDE_ERROR_DETAILS") == "true" {
        handlers.SetHideErrorDetails(true)
    }
    
    // Create domain services
    productService := services.NewProductService(db).
        WithLogger(logger).
        Build()
    
    categoryService := services.NewCategoryService(db).Build()
    
    // Build router with ogen server
    handler, err := handlers.NewRouter(productService).
        WithCategoryService(categoryService).
        WithMiddlewares(handlers.DefaultMiddlewares()...).
        Build()
    if err != nil {
        log.Fatal(err)
    }
    
    http.ListenAndServe(":8080", handler)
}
```

#### 7. Use as Go Package
External applications can import and use the service directly:

```go
// In another application
import "yourapp/services"

// Use domain service directly
productSvc := services.NewProductService(db).Build()
product, err := productSvc.Create(ctx, &api.CreateProductReq{Name: "Test"})

// Or use OgenHandler for full API compatibility
handler := services.NewOgenHandler().
    WithProductService(productSvc).
    Build()
product, err := handler.CreateProduct(ctx, &api.CreateProductReq{Name: "Test"})
```

#### 8. Update
Regenerate code whenever OpenAPI spec changes:
// turbo
```bash
go generate ./...
```

### Key Points

- **CRITICAL**: `OgenHandler` in services package implements ogen `Handler` interface and delegates to domain services
- **CRITICAL**: Domain services contain business logic, `OgenHandler` handles nil-checks and delegation
- **CRITICAL**: Router builder in `handlers/` creates ogen server with `ErrorHandler` and middlewares
- Services are reusable Go packages with OpenAPI-defined interfaces
- ogen handles HTTP routing, request parsing, response serialization
- Builder pattern for dependency injection: `NewService(db).WithLogger(log).Build()`
- Tests MUST use generated structs (NO `map[string]interface{}`)
- Tests MUST compare structs using `cmp.Diff()`
- ogen provides built-in validation based on OpenAPI schema
- ErrorHandler maps service sentinel errors to user-friendly HTTP responses

### AI Agent Requirements

- MUST check if OpenAPI spec exists in `api/openapi/` before generating code
- MUST install `ogen` if not available: `go install github.com/ogen-go/ogen/cmd/ogen@latest`
- MUST regenerate code whenever OpenAPI files change
- MUST create `services/ogen_handler.go` that implements `api.Handler` and delegates to domain services
- MUST create domain services with interfaces (e.g., `ProductService`) and implementations
- MUST create `handlers/routes.go` with router builder that creates ogen server with `ErrorHandler`
- MUST use builder pattern for both `OgenHandler` and domain services
- MUST use generated request/response types (NO manual struct definitions)
- Services MUST be in public packages (NOT `internal/`) for reusability
- ErrorHandler MUST map service sentinel errors to user-friendly responses
- NEVER expose internal error details in production (use `HIDE_ERROR_DETAILS=true`)

## Testing with ogen Server

### Testing Through ogen Server (Principle SERVEHTTP_TESTING)

```go
func TestProductCreate(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    
    // Create domain service
    productService := services.NewProductService(db).Build()
    
    // Build router with ogen server
    handler, err := handlers.NewRouter(productService).Build()
    if err != nil {
        t.Fatal(err)
    }
    
    // Test through ogen server ServeHTTP
    req := httptest.NewRequest("POST", "/api/v1/products", body)
    req.Header.Set("Content-Type", "application/json")
    rec := httptest.NewRecorder()
    
    handler.ServeHTTP(rec, req)  // ✅ Tests complete HTTP stack via ogen
    
    // Assertions...
}
```

### Why ogen Server (Not Manual Routes)

- ogen handles request validation based on OpenAPI schema
- ogen handles response serialization
- ogen handles path parameter extraction
- Tests validate the complete API contract
- No manual route registration that can drift from spec

### Testing AI Agent Requirements

- Tests MUST go through ogen server ServeHTTP (NOT individual service methods)
- Tests MUST use ogen-generated request/response types
- Tests MUST set `Content-Type: application/json` header
- Build router with `handlers.NewRouter().Build()` for tests

### Error Handling AI Agent Requirements

- MUST read the `details` field when debugging API errors
- MUST NOT assume `details` is always present (production hides it)
- SHOULD suggest setting `HIDE_ERROR_DETAILS=true` for production
- Tests MUST use `ErrorCodeMapping` definitions (NOT literal strings)
- Error codes MUST use ogen-generated `api.ErrorResponse` type (NOT custom structs)
- ALL errors (sentinel + HTTP codes) MUST have test cases

### See Also

- `/theplant.integration-test` - Testing ogen services with real database
