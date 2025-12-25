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

**CRITICAL**: The `services/` package implements the ogen-generated `Handler` interface directly. This allows:
- Services to be imported and used as Go packages by external applications
- OpenAPI spec defines the service contract (interface)
- No separate `handlers/` package needed for HTTP logic - ogen handles routing/serialization

```
api/
├── openapi/           # OpenAPI specifications (SINGLE SOURCE OF TRUTH)
│   └── <domain>.yaml
└── gen/<domain>/      # ogen generated code
    ├── gen.go         # go:generate directive
    └── oas_*.go       # generated types, server, client

services/              # PUBLIC - implements ogen Handler interface
├── <domain>_service.go  # Implements api/gen/<domain>.Handler
├── errors.go         # Sentinel errors
└── migrations.go     # AutoMigrate() for external apps

handlers/              # Server setup with error handling
├── server.go         # NewServer() wrapper with ErrorHandler
├── error_handler.go  # ogen ErrorHandler implementation
└── error_codes.go    # Error code definitions

internal/
├── models/           # GORM models (internal only)
└── config/
```

## Execution Steps

### Prerequisites

Install ogen if not available:
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

#### 4. Implement Handler in Services Package
**CRITICAL**: Services implement the ogen `Handler` interface directly (NOT in handlers package):

```go
// services/product_service.go
package services

import (
    "context"
    
    api "yourapp/api/gen/product"
    "yourapp/internal/models"
    "gorm.io/gorm"
)

// ProductService implements api.Handler interface
type ProductService struct {
    db     *gorm.DB
    logger Logger  // Optional
    cache  Cache   // Optional
}

// Ensure ProductService implements the generated Handler interface
var _ api.Handler = (*ProductService)(nil)

// Builder pattern for dependency injection
type productServiceBuilder struct {
    db     *gorm.DB
    logger Logger
    cache  Cache
}

func NewProductService(db *gorm.DB) *productServiceBuilder {
    return &productServiceBuilder{db: db}
}

func (b *productServiceBuilder) WithLogger(logger Logger) *productServiceBuilder {
    b.logger = logger
    return b
}

func (b *productServiceBuilder) WithCache(cache Cache) *productServiceBuilder {
    b.cache = cache
    return b
}

func (b *productServiceBuilder) Build() *ProductService {
    return &ProductService{db: b.db, logger: b.logger, cache: b.cache}
}

// Implement ogen Handler interface methods
func (s *ProductService) CreateProduct(ctx context.Context, req *api.CreateProductReq) (*api.Product, error) {
    // Context cancellation check
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }
    
    // Business logic and validation
    product := &models.Product{Name: req.Name, SKU: req.Sku}
    
    if err := s.db.WithContext(ctx).Create(product).Error; err != nil {
        if isDuplicateKeyError(err) {
            return nil, fmt.Errorf("create product SKU %s: %w", req.Sku, ErrDuplicateSKU)
        }
        return nil, fmt.Errorf("create product: %w", err)
    }
    
    return &api.Product{ID: product.ID, Name: product.Name, Sku: product.SKU}, nil
}

func (s *ProductService) GetProduct(ctx context.Context, params api.GetProductParams) (*api.Product, error) {
    var product models.Product
    if err := s.db.WithContext(ctx).First(&product, "id = ?", params.ID).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("get product %s: %w", params.ID, ErrProductNotFound)
        }
        return nil, fmt.Errorf("query product: %w", err)
    }
    return &api.Product{ID: product.ID, Name: product.Name, Sku: product.SKU}, nil
}
```

#### 5. Setup Server with ErrorHandler

**CRITICAL**: Server setup should be in `handlers/` package to encapsulate error handling configuration. This prevents service errors from leaking to users.

Create `handlers/server.go`:

```go
package handlers

import (
    "net/http"
    
    api "yourapp/api/gen/product"
)

// NewServer creates an ogen server with proper error handling configured
func NewServer(h api.Handler) (*api.Server, error) {
    return api.NewServer(
        h,
        api.WithErrorHandler(OgenErrorHandler),
    )
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
    
    // Service implements ogen Handler interface
    service := services.NewProductService(db).
        WithLogger(logger).
        Build()
    
    // Create server via handlers package (includes ErrorHandler)
    server, err := handlers.NewServer(service)
    if err != nil {
        log.Fatal(err)
    }
    
    http.ListenAndServe(":8080", server)
}
```

#### 6. Use as Go Package
External applications can import and use the service directly:

```go
// In another application
import "yourapp/services"

svc := services.NewProductService(db).Build()
product, err := svc.CreateProduct(ctx, &api.CreateProductReq{Name: "Test"})
```

#### 7. Update
Regenerate code whenever OpenAPI spec changes:
// turbo
```bash
go generate ./...
```

### Key Points

- **CRITICAL**: Services implement ogen `Handler` interface (NEVER in handlers package)
- **CRITICAL**: Server setup with `ErrorHandler` should be in `handlers/` package
- Services are reusable Go packages with OpenAPI-defined interfaces
- ogen handles HTTP routing, request parsing, response serialization
- Services contain business logic, validation, and database operations
- Builder pattern for dependency injection: `NewService(db).WithLogger(log).Build()`
- Tests MUST use generated structs (NO `map[string]interface{}`)
- Tests MUST compare structs using `cmp.Diff()`
- ogen provides built-in validation based on OpenAPI schema
- ErrorHandler maps service sentinel errors to user-friendly HTTP responses

### AI Agent Requirements

- MUST check if OpenAPI spec exists in `api/openapi/` before generating code
- MUST install `ogen` if not available: `go install github.com/ogen-go/ogen/cmd/ogen@latest`
- MUST regenerate code whenever OpenAPI files change
- MUST implement the generated `Handler` interface in `services/` package (NEVER in handlers/)
- MUST create `handlers/server.go` with `NewServer()` that configures `ErrorHandler`
- MUST use builder pattern for service construction
- MUST use generated request/response types (NO manual struct definitions)
- Services MUST be in public packages (NOT `internal/`) for reusability
- ErrorHandler MUST map service sentinel errors to user-friendly responses
- NEVER expose internal error details in production (use `HIDE_ERROR_DETAILS=true`)

### See Also

- `/theplant.errors` - Error handling with sentinel errors and HTTP mapping
- `/theplant.integration-test` - Testing ogen services with real database
