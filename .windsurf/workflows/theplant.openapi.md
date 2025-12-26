---
description: Generate Go types and server interfaces from OpenAPI specifications using ogen, with services implementing the Handler interface.
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
├── error_codes.go    # Error code definitions
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

Create `handlers/error_handler.go` (see `/theplant.errors` for full implementation):

```go
package handlers

import (
    "context"
    "encoding/json"
    "errors"
    "net/http"
    
    "yourapp/services"
)

// OgenErrorHandler maps service errors to user-friendly HTTP responses
func OgenErrorHandler(ctx context.Context, w http.ResponseWriter, r *http.Request, err error) {
    errCode := mapServiceError(err)
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(errCode.HTTPStatus)
    
    resp := ErrorResponse{
        Code:    errCode.Code,
        Message: errCode.Message,
    }
    
    // Include details in development only
    if !hideErrorDetails && err != nil {
        resp.Details = err.Error()
    }
    
    json.NewEncoder(w).Encode(resp)
}

func mapServiceError(err error) ErrorCode {
    // Check context errors
    if errors.Is(err, context.Canceled) {
        return Errors.RequestCancelled
    }
    if errors.Is(err, context.DeadlineExceeded) {
        return Errors.RequestTimeout
    }
    
    // Check service sentinel errors
    for _, errCode := range AllErrors() {
        if errCode.ServiceErr != nil && errors.Is(err, errCode.ServiceErr) {
            return errCode
        }
    }
    
    return Errors.InternalError
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

### See Also

- `/theplant.errors` - Error handling with sentinel errors and HTTP mapping
- `/theplant.integration-test` - Testing ogen services with real database
