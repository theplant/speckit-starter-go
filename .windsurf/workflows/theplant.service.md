---
description: Create services following the builder pattern and dependency injection principles from the constitution.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Create services following Principle SERVICE_ARCHITECTURE: Service Layer Architecture (Dependency Injection).

## Service Layer Architecture

### Core Requirements

- Business logic MUST be Go interfaces (service layer)
- Services MUST NOT depend on HTTP types (only `context.Context` allowed)
- Handlers MUST be thin wrappers delegating to services
- **ALL validation MUST be in services** (handlers MUST NOT validate)
- Services MUST use builder pattern: `NewService(db).WithLogger(log).Build()`
- Service methods MUST use protobuf/generated structs (NO primitives)

### Service Interface Definition

```go
// ✅ CORRECT: Use protobuf structs for ALL parameters
type ProductService interface {
    Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error)
    Get(ctx context.Context, req *pb.GetProductRequest) (*pb.Product, error)
    List(ctx context.Context, req *pb.ListProductsRequest) (*pb.ListProductsResponse, error)
}

// ❌ WRONG: NO primitives, NO maps
type ProductService interface {
    Create(ctx context.Context, name, sku string) (*pb.Product, error)  // NO!
    Get(ctx context.Context, id string) (*pb.Product, error)            // NO!
}
```

### Builder Pattern Implementation

```go
type productService struct {
    db     *gorm.DB
    logger Logger  // Optional
    cache  Cache   // Optional
}

// Unexported builder
type productServiceBuilder struct {
    db     *gorm.DB
    logger Logger
    cache  Cache
}

// Constructor: REQUIRED params only
func NewProductService(db *gorm.DB) *productServiceBuilder {
    return &productServiceBuilder{db: db}
}

// Chainable optional params
func (b *productServiceBuilder) WithLogger(logger Logger) *productServiceBuilder {
    b.logger = logger
    return b
}

func (b *productServiceBuilder) WithCache(cache Cache) *productServiceBuilder {
    b.cache = cache
    return b
}

func (b *productServiceBuilder) Build() ProductService {
    return &productService{db: b.db, logger: b.logger, cache: b.cache}
}
```

### Service Method Implementation

```go
func (s *productService) Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    // Principle CONTEXT_AWARE: Check cancellation before expensive operations
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }
    
    // Principle ERROR_HANDLING: Validate and return sentinel errors
    // ALL validation in service (handler does NOT validate)
    if req.Name == "" {
        return nil, fmt.Errorf("product name: %w", ErrMissingRequired)
    }
    
    product := &Product{Name: req.Name, SKU: req.Sku}
    
    // Principle CONTEXT_AWARE: Use context-aware database operations
    if err := s.db.WithContext(ctx).Create(product).Error; err != nil {
        // Principle ERROR_HANDLING: Wrap errors with context
        if isDuplicateKeyError(err) {
            return nil, fmt.Errorf("create product SKU %s: %w", req.Sku, ErrDuplicateSKU)
        }
        return nil, fmt.Errorf("create product: %w", err)
    }
    
    // Principle SCHEMA_FIRST: Return protobuf type
    return &pb.Product{Id: product.ID, Name: product.Name, Sku: product.SKU}, nil
}
```

### Thin HTTP Handler

```go
// Handler is thin wrapper: extract params, call service, write response
// NO validation in handler - service validates
func (h *ProductHandler) Get(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    id := r.PathValue("id")  // Extract path parameter (Go 1.22+)
    
    // Service validates req.Id (handler does NOT check if empty)
    product, err := h.service.Get(ctx, &pb.GetProductRequest{Id: id})
    if err != nil {
        HandleServiceError(w, err)  // Automatic error mapping
        return
    }
    
    RespondWithProto(w, http.StatusOK, product)
}
```

### Package Structure

```
github.com/theplant/myapp/
├── api/
│   ├── openapi/      # PUBLIC - OpenAPI specifications
│   ├── proto/        # PUBLIC - protobuf source files
│   └── gen/          # PUBLIC - generated code
├── services/         # PUBLIC - reusable by external apps
│   ├── product_service.go
│   └── errors.go
├── handlers/         # PUBLIC - reusable routing/HTTP logic
│   ├── product_handler.go
│   └── error_codes.go
└── internal/         # INTERNAL - implementation only
    ├── models/       # ✅ OK (services return protobuf, not models)
    └── config/
```

### Usage Examples

```go
// Minimal (tests)
svc := NewProductService(db).Build()

// Production (with logging)
svc := NewProductService(db).WithLogger(log).WithCache(cache).Build()
```

### AI Agent Requirements

- Services MUST use builder pattern with required params in constructor
- Service methods MUST use protobuf/generated structs (NO primitives)
- ALL validation MUST be in services (handlers are thin wrappers)
- Services MUST be in public packages (NOT `internal/`) for reusability
- Services MUST NOT depend on HTTP types

### See Also

- `/theplant.routes` - Setup HTTP routing for services
- `/theplant.errors` - Implement error handling strategy
