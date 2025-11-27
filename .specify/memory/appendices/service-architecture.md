# Service Architecture Appendix

**Part of**: [Go Project Constitution](../constitution.md)  
**Covers**: Principle XI

This appendix provides detailed implementation guidelines for service layer architecture, dependency injection, package structure, and middleware patterns.

---

## System Architecture

### XI. Service Layer Architecture (Dependency Injection)

- Business logic MUST be implemented as Go interfaces (service layer)
- Services MUST NOT depend on HTTP types (`http.Request`, `http.ResponseWriter`, `context.Context` is allowed)
- HTTP handlers MUST be thin wrappers that call service methods
- Services MUST accept all inputs as method parameters (no HTTP request parsing in services)
- External dependencies (database, logger, cache, etc.) MUST be injected via constructor
- Services MUST be exported and usable as normal Go packages
- Service interfaces MUST be defined in the same package as implementation
- Multiple applications MUST be able to share the same service instances
- **Services MUST NOT be in `internal/` packages** - they are intended for use by external Go applications
- **Handlers MUST NOT be in `internal/` packages** - they should be reusable across applications
- Use `internal/` ONLY for code that is truly internal implementation details, not part of the public API

**Rationale**: Separating business logic from HTTP transport enables code reuse across multiple contexts (HTTP APIs, gRPC services, CLI tools, background workers, embedded usage in other Go apps). Dependency injection allows applications to share expensive resources like database connection pools and caches. This architecture makes services testable without HTTP layer overhead and allows the package to be imported and used as a library in other Go applications. **CRITICAL**: Go's `internal/` package visibility rules prevent external applications from importing packages within `internal/`. Therefore, services and handlers must be in public packages (e.g., `services/`, `handlers/`) to enable cross-application reuse as described in the usage examples. Handlers in public packages allow external applications to reuse routing configurations and HTTP handling logic.

**Architecture Layers**:
```
HTTP Handler (thin) → Service Interface (business logic) → Repository (data access)
```

### Service Method Parameters: Protobuf Structs Only

**CRITICAL REQUIREMENT**: All service method parameters MUST be protobuf structs. This is non-negotiable.

**Rule**: Service methods MUST accept protobuf request structs and return protobuf response structs. NO primitive types, NO maps, NO slices as direct parameters.

**✅ CORRECT - Protobuf Struct Parameters**:
```go
// Service interface - ALL parameters are protobuf structs
type ProductService interface {
    Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error)
    Get(ctx context.Context, req *pb.GetProductRequest) (*pb.Product, error)
    List(ctx context.Context, req *pb.ListProductsRequest) (*pb.ListProductsResponse, error)
    Update(ctx context.Context, req *pb.UpdateProductRequest) (*pb.Product, error)
    Delete(ctx context.Context, req *pb.DeleteProductRequest) (*pb.DeleteProductResponse, error)
}

// Protobuf definitions
message CreateProductRequest {
    string name = 1;
    string sku = 2;
    string description = 3;
}

message GetProductRequest {
    string id = 1;
}

message ListProductsRequest {
    PaginationRequest pagination = 1;
    ProductFilter filter = 2;
}

message DeleteProductRequest {
    string id = 1;
}

message DeleteProductResponse {
    bool success = 1;
}
```

**❌ WRONG - Primitive Type Parameters**:
```go
// ❌ BAD: Primitive types as parameters
type ProductService interface {
    Create(ctx context.Context, name, sku, description string) (*pb.Product, error)  // NO!
    Get(ctx context.Context, id string) (*pb.Product, error)  // NO!
    List(ctx context.Context, limit, offset int) ([]*pb.Product, error)  // NO!
    Update(ctx context.Context, id string, updates map[string]interface{}) (*pb.Product, error)  // NO!
    Delete(ctx context.Context, id string) error  // NO!
}
```

**Rationale - Why Protobuf Structs Only**:

1. **API Contract**: Protobuf provides versioned, documented API contract
2. **Validation**: Built-in validation rules (required, min/max, pattern)
3. **Type Safety**: Compile-time checking prevents parameter mismatches
4. **Evolution**: Easy to add new fields without breaking existing code
5. **Cross-Language**: External apps in any language can use the same API
6. **Documentation**: Protobuf comments become API documentation
7. **Consistency**: Same types used by HTTP, gRPC, CLI, workers
8. **Serialization**: Protobuf handles JSON/binary serialization

**Why Not Primitive Types**:

```go
// ❌ Problem 1: No validation
func (s *service) Create(ctx context.Context, name, sku string) (*pb.Product, error) {
    // Must manually validate every parameter
    if name == "" {
        return nil, errors.New("name required")
    }
    if len(sku) < 3 {
        return nil, errors.New("SKU too short")
    }
    // ... validation code everywhere
}

// ✅ Solution: Protobuf validation
message CreateProductRequest {
    string name = 1 [(validate.rules).string.min_len = 1];
    string sku = 2 [(validate.rules).string = {min_len: 3, pattern: "^[A-Z0-9-]+$"}];
}
// Validation happens automatically before service is called

// ❌ Problem 2: Cannot add optional fields
func (s *service) Create(ctx context.Context, name, sku string) (*pb.Product, error)
// Adding "description" parameter breaks all callers!

// ✅ Solution: Protobuf optional fields
message CreateProductRequest {
    string name = 1;
    string sku = 2;
    optional string description = 3;  // Can add without breaking callers
}

// ❌ Problem 3: No API documentation
func (s *service) List(ctx context.Context, limit, offset int) ([]*pb.Product, error)
// What's the maximum limit? What happens with negative offset? Undocumented!

// ✅ Solution: Protobuf with comments
message ListProductsRequest {
    // Maximum 100 items per page
    int32 page_size = 1 [(validate.rules).int32 = {gte: 1, lte: 100}];
    // Page number starting from 1
    int32 page = 2 [(validate.rules).int32.gte = 1];
}

// ❌ Problem 4: Maps are untyped
func (s *service) Update(ctx context.Context, id string, updates map[string]interface{}) (*pb.Product, error)
// What keys are valid? What value types? Runtime errors!

// ✅ Solution: Protobuf typed fields
message UpdateProductRequest {
    string id = 1;
    optional string name = 2;
    optional string description = 3;
    optional double price = 4;
}
```

**Simple ID Lookup Exception**:

Even for simple ID lookups, use protobuf structs:

```go
// ✅ CORRECT
type ProductService interface {
    Get(ctx context.Context, req *pb.GetProductRequest) (*pb.Product, error)
}

message GetProductRequest {
    string id = 1 [(validate.rules).string.uuid = true];
}

// ❌ WRONG
type ProductService interface {
    Get(ctx context.Context, id string) (*pb.Product, error)
}
```

**Why?** Because later you might need to add:
- `include_deleted bool` - show soft-deleted products
- `expand []string` - which related objects to load
- `fields_mask` - which fields to return

With protobuf struct, you just add optional fields. With primitive parameter, you break the API.

**Return Types**:

Services MUST return protobuf types OR protobuf + error:

```go
// ✅ CORRECT return types
Create(ctx, req) (*pb.Product, error)                    // Single protobuf + error
List(ctx, req) (*pb.ListProductsResponse, error)         // Protobuf response with list
Delete(ctx, req) (*pb.DeleteProductResponse, error)      // Protobuf response for confirmation

// ❌ WRONG return types
Create(ctx, req) (*Product, error)                       // Internal model (not protobuf)
List(ctx, req) ([]*pb.Product, *Pagination, error)       // Multiple returns (use protobuf wrapper)
Delete(ctx, req) error                                   // No protobuf response
```

**List/Pagination Pattern**:

```go
// ❌ WRONG: Multiple return values
List(ctx context.Context, req *pb.ListProductsRequest) ([]*pb.Product, *Pagination, error)

// ✅ CORRECT: Single protobuf response
List(ctx context.Context, req *pb.ListProductsRequest) (*pb.ListProductsResponse, error)

message ListProductsResponse {
    repeated Product products = 1;
    PaginationResponse pagination = 2;
}

message PaginationResponse {
    int32 total_items = 1;
    int32 total_pages = 2;
    int32 current_page = 3;
    int32 page_size = 4;
}
```

**Enforcement**:

This requirement is enforced at:
1. **Code Review**: Reviewers MUST reject service methods with primitive parameters
2. **Interface Definition**: Service interfaces define the contract
3. **External Apps**: Cannot call services without protobuf types
4. **Protoc Generation**: Protobuf files are generated code (cannot bypass)



**Package Structure for Reusability**:

```go
// ✅ CORRECT: Services in public package (reusable)
github.com/theplant/myapp/
├── services/          // PUBLIC - can be imported by other apps
│   ├── product_service.go
│   ├── order_service.go
│   └── errors.go      // Sentinel errors
├── handlers/          // PUBLIC - can be reused by other apps (REQUIRED)
│   ├── product_handler.go
│   └── error_codes.go
├── api/gen/pim/v1/    // PUBLIC - protobuf generated (MUST be importable)
│   ├── product.pb.go  // Returned by ProductService.Create()
│   └── template.pb.go
└── internal/          // INTERNAL - implementation details only
    ├── models/        // ✅ CAN be internal (services return protobuf, not models)
    │   └── product.go // Only used inside services for database mapping
    ├── database/      // Connection helpers (app-specific)
    ├── middleware/    // Logging, CORS (app-specific)
    └── config/        // Config loading (not exposed)

// ❌ WRONG: Services and handlers in internal (NOT reusable)
github.com/theplant/myapp/
└── internal/
    ├── services/      // WRONG! Cannot be imported externally
    │   └── product_service.go
    └── handlers/      // WRONG! Handlers must be public for reusability
        └── product_handler.go

// External app trying to use services:
import "github.com/theplant/myapp/internal/services"  // ❌ FAILS! Cannot import internal
```

**Why Models CAN Stay Internal** (Services Return Protobuf):
```go
// ✅ CORRECT: Services return protobuf (public), use models internally
// services/product_service.go (PUBLIC package)
package services

import (
    "yourapp/internal/models"  // ✅ OK! Internal use within service
    pb "yourapp/api/gen/pim/v1"  // ✅ PUBLIC protobuf types
    "gorm.io/gorm"
)

type ProductService interface {
    Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error)
    //                                                          ^^^^^^^^^^^^^
    //                                                          PUBLIC protobuf type!
}

type productService struct {
    db *gorm.DB
}

func (s *productService) Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    // Use internal models for database
    product := &models.Product{  // ✅ Internal model (private)
        Name: req.Name,
        SKU:  req.Sku,
    }
    
    s.db.Create(product)  // Save GORM model
    
    // Return protobuf (public type)
    return &pb.Product{  // ✅ Public protobuf
        Id:   product.ID,
        Name: product.Name,
        Sku:  product.SKU,
    }, nil
}

// External app using service:
import (
    "yourapp/services"  // ✅ Can import
    pb "yourapp/api/gen/pim/v1"  // ✅ Can import protobuf
)

svc := services.NewProductService(db)
product, err := svc.Create(ctx, &pb.CreateProductRequest{...})
if err != nil {
    log.Fatal(err)
}
// product is *pb.Product (public protobuf) - external app can use it! ✅
// models.Product is never exposed - stays internal ✅
```

**Key Insight**: Because services follow Principle VI (Protobuf Data Structures), they return protobuf types, not GORM models. Models are **only used internally** for database mapping and **never exposed** in the public API. Therefore, models CAN safely remain in `internal/models/`.

**When to Use internal/**:
- ✅ Database connection helpers (not part of public API)
- ✅ **Middleware** (application-specific - logging format, CORS policies, etc.)
- ✅ **Models** (internal data structures - services return protobuf, not models)
- ✅ Configuration loaders (implementation detail)
- ✅ Private utilities (not meant for external use)
- ❌ **Services** (MUST be public - promised reusability)
- ❌ **Handlers** (MUST be public - enables cross-application handler reuse)
- ❌ **Protobuf generated code** (api/gen/ - needed by external apps to call services)

**IMPORTANT - Protobuf Return Types**: Services return **protobuf types** (`*pb.Product`, `*pb.ProductTemplate`), NOT GORM models. This means:
- ✅ Models CAN stay in `internal/models/` (only used internally by services)
- ✅ Protobuf types (`api/gen/pim/v1/`) MUST be importable (already generated in non-internal location)
- ✅ External apps only need to import services and protobuf packages, not models
- ✅ Go's compiler is happy - public services return public types (protobuf)

**CRITICAL - Database Migrations for External Apps**: External applications using services need to run database migrations, but cannot import internal models. **Solution**: Services MUST export an `AutoMigrate()` function that runs migrations on behalf of external apps:

```go
// services/migrations.go (PUBLIC package)
package services

import (
    "gorm.io/gorm"
    "yourapp/internal/models"  // Internal use within service package
)

// AutoMigrate runs all necessary database migrations for services
// External apps MUST call this before using services
func AutoMigrate(db *gorm.DB) error {
    return db.AutoMigrate(
        &models.ProductTemplate{},
        &models.Product{},
        &models.ProductPricing{},
        &models.ProductInventory{},
        &models.ProductVariant{},
        &models.VariantPricing{},
        &models.VariantInventory{},
        &models.MediaFile{},
    )
}

// External app usage:
import "yourapp/services"

db, err := gorm.Open(...)
if err != nil {
    log.Fatal(err)
}
if err := services.AutoMigrate(db); err != nil {  // ✅ Migrates schema
    log.Fatal(err)
}
svc := services.NewProductService(db)  // ✅ Ready to use
```

This approach:
- ✅ Keeps models internal (encapsulation)
- ✅ Allows external apps to migrate schema
- ✅ Services control their own schema requirements
- ✅ No need to expose GORM model internals

**Middleware Rationale**: Middleware is application-specific implementation (logging format, CORS policies, recovery behavior). External apps reusing your handlers should apply their own middleware. Keep middleware internal unless you're building a middleware library.

**Example Service Interface**:
```go
// Service interface - pure business logic, no HTTP dependencies
type ProductService interface {
    Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error)
    Get(ctx context.Context, req *pb.GetProductRequest) (*pb.Product, error)
    List(ctx context.Context, req *pb.ListProductsRequest) (*pb.ListProductsResponse, error)
    Update(ctx context.Context, req *pb.UpdateProductRequest) (*pb.Product, error)
    Delete(ctx context.Context, req *pb.DeleteProductRequest) (*pb.DeleteProductResponse, error)
}

// Service implementation with dependency injection
type productService struct {
    db     *gorm.DB
    logger Logger
    cache  Cache
}

// Builder for service configuration (unexported)
type productServiceBuilder struct {
    db     *gorm.DB
    logger Logger
    cache  Cache
}

// NewProductService creates a builder with REQUIRED parameters only
// Returns builder for fluent API configuration
func NewProductService(db *gorm.DB) *productServiceBuilder {
    return &productServiceBuilder{db: db}
}

// WithLogger sets optional logger (chainable)
func (b *productServiceBuilder) WithLogger(logger Logger) *productServiceBuilder {
    b.logger = logger
    return b
}

// WithCache sets optional cache (chainable)
func (b *productServiceBuilder) WithCache(cache Cache) *productServiceBuilder {
    b.cache = cache
    return b
}

// Build constructs the final service
func (b *productServiceBuilder) Build() ProductService {
    return &productService{
        db:     b.db,
        logger: b.logger,
        cache:  b.cache,
    }
}

// Service method - pure business logic, no HTTP types
func (s *productService) Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    // Validate business rules
    if req.Name == "" {
        return nil, errors.New("product name required")
    }
    
    // Create entity
    product := &Product{
        Name: req.Name,
        SKU:  req.Sku,
        Description: req.Description,
    }
    
    // Persist with transaction
    tx := s.db.WithContext(ctx).Begin()
    defer tx.Rollback()
    
    if err := tx.Create(product).Error; err != nil {
        if s.logger != nil {
            s.logger.Error("failed to create product", err)
        }
        return nil, err
    }
    
    if err := tx.Commit().Error; err != nil {
        return nil, err
    }
    
    // Convert to protobuf response
    return &pb.Product{
        Id:          product.ID,
        Name:        product.Name,
        Sku:         product.SKU,
        Description: product.Description,
    }, nil
}
```

**Builder Pattern Benefits**:

The builder pattern provides several key advantages:

1. **Clear Required vs Optional**: Database is required, logger and cache are optional
2. **Readable Configuration**: Chainable methods make intent clear
3. **Flexible Setup**: External apps can choose which dependencies to provide
4. **Future-Proof**: Easy to add new optional dependencies without breaking existing code
5. **Type Safety**: Compiler ensures required parameters are provided

**Usage Examples**:

```go
// Minimal setup (database only)
svc := NewProductService(db).Build()

// With logger
svc := NewProductService(db).
    WithLogger(logger).
    Build()

// Full configuration
svc := NewProductService(db).
    WithLogger(logger).
    WithCache(cache).
    Build()

// ❌ WRONG: Cannot forget required database parameter
svc := NewProductService().Build()  // Compilation error!
```

**Why Not Simple Constructor?**

```go
// ❌ Traditional approach - unclear what's optional
func NewProductService(db *gorm.DB, logger Logger, cache Cache) ProductService

// Problem 1: All parameters look required
// Problem 2: External apps must pass nil for unused dependencies
svc := NewProductService(db, nil, nil)  // Unclear intent

// Problem 3: Adding new optional parameter breaks all existing code
func NewProductService(db *gorm.DB, logger Logger, cache Cache, metrics Metrics) ProductService
// All callers must be updated!

// ✅ Builder pattern - clear and flexible
svc := NewProductService(db).Build()  // Still works with new optional parameters
```

**Service Customization with Middleware Pattern**:

Services MAY expose middleware functions for external extensibility. Middleware provides full control: run code before/after operations, abort execution, or transform inputs/outputs.

**Note**: Middleware pattern naturally extends the builder pattern shown above. The same builder can support both optional dependencies (logger, cache) and optional middleware (validation, audit, notifications).

**Middleware Pattern**:
```go
// CreateMiddleware wraps the Create operation, providing full control
// Parameters:
//   - ctx: Request context
//   - req: Creation request
//   - next: Function to call core Create logic (can be skipped)
// Returns: (*pb.Product, error)
// 
// Middleware can:
//   - Run code before core logic
//   - Decide whether to call next() or abort
//   - Run code after core logic
//   - Transform inputs or outputs
//   - Handle or wrap errors
type CreateMiddleware func(
    ctx context.Context,
    req *pb.CreateProductRequest,
    next func(context.Context, *pb.CreateProductRequest) (*pb.Product, error),
) (*pb.Product, error)

// Service struct with pre-built middleware handler
type productService struct {
    db            *gorm.DB
    createHandler func(context.Context, *pb.CreateProductRequest) (*pb.Product, error)  // Pre-built chain
}

// serviceBuilder provides fluent API for service configuration
// Note: Unexported type - external apps don't need to know about it
type serviceBuilder struct {
    db                *gorm.DB
    createMiddlewares []CreateMiddleware
    logger            Logger
    cache             Cache
}

// NewProductService creates a builder with REQUIRED parameters only
// Returns unexported builder with exported methods for fluent API
// Use With* methods to add optional configuration, then call Build()
func NewProductService(db *gorm.DB) *serviceBuilder {
    return &serviceBuilder{db: db}
}

// WithCreateMiddleware adds middleware to Create operation (chainable)
func (b *serviceBuilder) WithCreateMiddleware(mw CreateMiddleware) *serviceBuilder {
    b.createMiddlewares = append(b.createMiddlewares, mw)
    return b  // Return builder for chaining
}

// WithLogger sets optional logger (chainable)
func (b *serviceBuilder) WithLogger(logger Logger) *serviceBuilder {
    b.logger = logger
    return b
}

// WithCache sets optional cache (chainable)
func (b *serviceBuilder) WithCache(cache Cache) *serviceBuilder {
    b.cache = cache
    return b
}

// Build constructs the service and builds middleware chain ONCE
func (b *serviceBuilder) Build() ProductService {
    svc := &productService{
        db:     b.db,
        logger: b.logger,
        cache:  b.cache,
    }
    
    // Build middleware chain once (not on every call!)
    handler := svc.coreCreate
    
    // Wrap with middleware (reverse order for correct execution)
    for i := len(b.createMiddlewares) - 1; i >= 0; i-- {
        mw := b.createMiddlewares[i]
        next := handler
        handler = func(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
            return mw(ctx, req, next)
        }
    }
    
    svc.createHandler = handler
    return svc
}

// Core create logic (private)
func (s *productService) coreCreate(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    // Standard validation
    if req.Name == "" {
        return nil, fmt.Errorf("name: %w", ErrMissingRequired)
    }
    
    // Create in database
    product := &Product{Name: req.Name, SKU: req.Sku}
    if err := s.db.WithContext(ctx).Create(product).Error; err != nil {
        return nil, fmt.Errorf("create product: %w", err)
    }
    
    return toProto(product), nil
}

// Public Create method simply executes pre-built chain
func (s *productService) Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    // Execute pre-built middleware chain (built once in constructor)
    return s.createHandler(ctx, req)
}
```

**External App Using Middleware**:
```go
import "yourapp/services"

func main() {
    db, err := gorm.Open(...)
    if err != nil {
        log.Fatal(err)
    }
    
    // Middleware 1: Validation
    validateSKU := func(
        ctx context.Context,
        req *pb.CreateProductRequest,
        next func(context.Context, *pb.CreateProductRequest) (*pb.Product, error),
    ) (*pb.Product, error) {
        // Run before core logic
        if !strings.HasPrefix(req.Sku, "ACME-") {
            return nil, errors.New("SKU must start with ACME-")
        }
        
        // Call next middleware or core logic
        return next(ctx, req)
    }
    
    // Middleware 2: Notification
    notifySlack := func(
        ctx context.Context,
        req *pb.CreateProductRequest,
        next func(context.Context, *pb.CreateProductRequest) (*pb.Product, error),
    ) (*pb.Product, error) {
        // Call next first
        product, err := next(ctx, req)
        if err != nil {
            return nil, err
        }
        
        // Run after core logic
        sendSlackNotification(fmt.Sprintf("Created: %s", product.Name))
        
        return product, nil
    }
    
    // Middleware 3: Audit logging
    auditLog := func(
        ctx context.Context,
        req *pb.CreateProductRequest,
        next func(context.Context, *pb.CreateProductRequest) (*pb.Product, error),
    ) (*pb.Product, error) {
        // Run before
        startTime := time.Now()
        userID := getUserIDFromContext(ctx)
        
        // Call next
        product, err := next(ctx, req)
        
        // Run after (even if error)
        auditDB.Log(&AuditEntry{
            Action:   "product.create",
            UserID:   userID,
            Duration: time.Since(startTime),
            Success:  err == nil,
            Details:  fmt.Sprintf("SKU: %s", req.Sku),
        })
        
        return product, err
    }
    
    // Create service with fluent API (chainable configuration)
    productSvc := services.NewProductService(db).  // Required params only
        WithCreateMiddleware(validateSKU).         // Optional: validation
        WithCreateMiddleware(auditLog).            // Optional: audit
        WithCreateMiddleware(notifySlack).         // Optional: notification
        WithLogger(logger).                        // Optional: logger
        WithCache(cache).                          // Optional: cache
        Build()                                    // Builds middleware chain ONCE
    
    // Use normally - middleware chain executes automatically
    product, err := productSvc.Create(ctx, &pb.CreateProductRequest{
        Name: "Widget",
        Sku:  "ACME-001",
    })
    
    // Execution flow:
    // 1. validateSKU runs → validates → calls next
    // 2. auditLog runs (before) → calls next
    // 3. notifySlack runs → calls next
    // 4. coreCreate runs → creates product
    // 5. notifySlack runs (after) → sends notification
    // 6. auditLog runs (after) → logs audit
    // 7. Return product
}
```

**Middleware Design Best Practices**:
- ✅ **Builder Pattern**: Constructor takes REQUIRED params (database), With* methods for OPTIONAL params (logger, cache, middleware)
- ✅ **Fluent API**: With* methods return builder for chaining
- ✅ **Build once**: Build() method constructs service and middleware chain
- ✅ Export middleware function type with clear documentation
- ✅ Middleware receives `next` function to call core logic
- ✅ Middleware can decide whether to call `next` (abort if needed)
- ✅ Chain middleware in correct order (first added executes first)
- ✅ Document execution order clearly
- ✅ Pass protobuf types (never internal models)
- ✅ Provide example middleware for common use cases
- ✅ Same builder handles both dependencies and middleware
- ❌ Don't expose internal service state
- ❌ Don't rebuild middleware chain on every method call (inefficient)
- ❌ Don't make optional dependencies required parameters

**Common Middleware Use Cases**:
- **Authorization**: Check permissions before allowing operation
- **Validation**: Custom business rules that can reject operations
- **Rate limiting**: Throttle operations per user/client
- **Caching**: Check cache before database, update after
- **Audit logging**: Track all operations with timing
- **Notifications**: Send alerts on specific operations
- **Transformations**: Modify requests or responses
- **Error handling**: Wrap or transform errors
- **Retry logic**: Retry failed operations
- **Circuit breakers**: Fail fast when downstream is down


**HTTP Handler (Thin Wrapper)**:
```go
// HTTP handler delegates to service
type ProductHandler struct {
    service ProductService // Injected dependency
}

func NewProductHandler(service ProductService) *ProductHandler {
    return &ProductHandler{service: service}
}

func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    // Extract trace span (HTTP concern)
    span, ctx := opentracing.StartSpanFromContext(r.Context(), "POST /api/products")
    defer span.Finish()
    
    // Parse HTTP request (HTTP concern)
    var req pb.CreateProductRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request", 400)
        return
    }
    
    // Delegate to service (business logic)
    product, err := h.service.Create(ctx, &req)
    if err != nil {
        span.SetTag("error", true)
        http.Error(w, err.Error(), 500)
        return
    }
    
    // Format HTTP response (HTTP concern)
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(product)
}

// Example: Setting up handler with service
func SetupProductHandler(db *gorm.DB, logger Logger, cache Cache) *ProductHandler {
    // Use builder pattern to create service
    service := NewProductService(db).
        WithLogger(logger).
        WithCache(cache).
        Build()
    
    return NewProductHandler(service)
}
```

**Usage in Other Go Applications**:
```go
// Application 1: HTTP API server
func main() {
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatal(err)
    }
    logger := NewLogger()
    cache := NewCache()
    
    // Create service using builder pattern
    productService := NewProductService(db).
        WithLogger(logger).
        WithCache(cache).
        Build()
    
    // Use in HTTP handlers
    handler := NewProductHandler(productService)
    http.HandleFunc("/api/products", handler.Create)
    http.ListenAndServe(":8080", nil)
}

// Application 2: Background worker (shares same db/logger)
func main() {
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatal(err)
    }
    
    // Run migrations first (REQUIRED for external apps)
    if err := services.AutoMigrate(db); err != nil {
        log.Fatalf("Migration failed: %v", err)
    }
    
    logger := NewLogger()
    cache := NewCache()
    
    // Create service using builder pattern
    productService := NewProductService(db).
        WithLogger(logger).
        WithCache(cache).
        Build()
    
    // Use directly without HTTP layer
    ctx := context.Background()
    products, err := productService.List(ctx, 100, 0)
    if err != nil {
        log.Fatal(err)
    }
    for _, p := range products {
        // Process products...
    }
}

// Application 3: Embedded in larger application
import "github.com/theplant/apidemo2/services"  // PUBLIC package (not internal/)

func processOrders() {
    // Run migrations once at app startup
    services.AutoMigrate(sharedDB)
    
    // Create service using builder pattern
    productSvc := services.NewProductService(sharedDB).
        WithLogger(sharedLogger).
        WithCache(sharedCache).
        Build()
    
    product, err := productSvc.Get(ctx, productID)
    // Use product in your business logic
}

// Application 4: Minimal setup (database only)
func minimalExample() {
    db, _ := gorm.Open(...)
    
    // Builder pattern allows omitting optional dependencies
    productSvc := services.NewProductService(db).Build()  // No logger, no cache
    
    product, err := productSvc.Get(ctx, productID)
    // Works fine without logger/cache
}

// Note: Services in internal/services would FAIL:
// import "github.com/theplant/apidemo2/internal/services"  // ❌ Cannot import internal
```




---

## External Application Usage Examples

This section provides complete, working examples of how external Go applications can import and use services as libraries.

### Complete Example: Using Services in External Go Application

This example demonstrates a full external application that imports and uses the services.

**Step 1: Import the Module**

```go
// external-app/go.mod
module github.com/myorg/external-app

go 1.21

require (
    github.com/theplant/yourapp/services v0.0.0  // Import the services
    github.com/theplant/yourapp/api/gen/pim/v1 v0.0.0  // Import protobuf types
    gorm.io/driver/postgres v1.5.0
    gorm.io/gorm v1.25.0
)
```

**Step 2: External Application Code**

```go
// external-app/main.go
package main

import (
    "context"
    "log"
    
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    
    // Import public packages
    "github.com/theplant/yourapp/services"
    pb "github.com/theplant/yourapp/api/gen/pim/v1"
)

func main() {
    log.Println("Starting external app using services...")
    
    // Step 1: Connect to YOUR database
    dsn := "host=localhost user=myapp password=myapp dbname=myapp_db port=5432"
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatalf("Failed to connect to database: %v", err)
    }
    log.Println("✅ Connected to database")
    
    // Step 2: Run migrations (REQUIRED)
    // This creates the necessary tables in YOUR database
    if err := services.AutoMigrate(db); err != nil {
        log.Fatalf("❌ Migration failed: %v", err)
    }
    log.Println("✅ Database schema migrated")
    
    // Step 3: Create services using builder pattern
    productService := services.NewProductService(db).Build()
    templateService := services.NewTemplateService(db).Build()
    
    ctx := context.Background()
    
    // Create a template
    template, err := templateService.Create(ctx, &pb.CreateTemplateRequest{
        Name: "Electronics",
        Attributes: []*pb.AttributeDefinition{
            {
                Name:     "Brand",
                Type:     pb.AttributeType_ATTRIBUTE_TYPE_TEXT,
                Required: true,
            },
            {
                Name:     "Color",
                Type:     pb.AttributeType_ATTRIBUTE_TYPE_LIST,
                Required: true,
                Options:  []string{"Black", "White", "Silver"},
            },
        },
    })
    if err != nil {
        log.Fatalf("Failed to create template: %v", err)
    }
    log.Printf("✅ Created template: %s (ID: %s)", template.Name, template.Id)
    
    // Create a product
    product, err := productService.Create(ctx, &pb.CreateProductRequest{
        TemplateId:       template.Id,
        Name:             "Laptop Pro",
        Sku:              "LAPTOP-001",
        Description:      "High-performance laptop",
        InitialListPrice: 1299.99,
        InitialStock:     50,
        AttributeValues: map[string]*pb.AttributeValue{
            "Brand": {
                Type:      pb.AttributeType_ATTRIBUTE_TYPE_TEXT,
                TextValue: "TechCorp",
            },
            "Color": {
                Type:      pb.AttributeType_ATTRIBUTE_TYPE_LIST,
                ListValue: []string{"Silver"},
            },
        },
        Status: pb.ProductStatus_PRODUCT_STATUS_ACTIVE,
    })
    if err != nil {
        log.Fatalf("Failed to create product: %v", err)
    }
    log.Printf("✅ Created product: %s (SKU: %s, ID: %s)", product.Name, product.Sku, product.Id)
    
    // List products
    products, pagination, err := productService.List(ctx, &pb.ListProductsRequest{
        Pagination: &pb.PaginationRequest{
            Page:     1,
            PageSize: 10,
        },
    })
    if err != nil {
        log.Fatalf("Failed to list products: %v", err)
    }
    log.Printf("✅ Found %d products (total: %d)", len(products), pagination.TotalItems)
    
    // Get a specific product
    getResp, err := productService.Get(ctx, &pb.GetProductRequest{
        Id: product.Id,
    })
    if err != nil {
        log.Fatalf("Failed to get product: %v", err)
    }
    log.Printf("✅ Retrieved product: %s", getResp.Product.Name)
    
    log.Println("🎉 External app successfully using services!")
}
```

**Expected Output**:
```
Starting external app using services...
✅ Connected to database
✅ Database schema migrated
✅ Created template: Electronics (ID: xxx)
✅ Created product: Laptop Pro (SKU: LAPTOP-001, ID: xxx)
✅ Found 1 products (total: 1)
✅ Retrieved product: Laptop Pro
🎉 External app successfully using services!
```

### Key Points for External Apps

**✅ What External Apps Can Import**:
```go
import (
    "github.com/theplant/yourapp/services"         // ✅ Public services
    "github.com/theplant/yourapp/handlers"         // ✅ Public handlers
    pb "github.com/theplant/yourapp/api/gen/pim/v1"  // ✅ Protobuf types
)
```

**❌ What External Apps CANNOT Import**:
```go
import (
    "github.com/theplant/yourapp/internal/models"      // ❌ Internal - won't compile
    "github.com/theplant/yourapp/internal/middleware"  // ❌ Internal - won't compile
    "github.com/theplant/yourapp/internal/database"    // ❌ Internal - won't compile
)
```

**✅ What External Apps Interact With**:
- **Protobuf types** (`*pb.Product`, `*pb.ProductTemplate`) - All service inputs and outputs
- **Service interfaces** - All business logic
- **Handlers** - Can reuse HTTP handlers with their own middleware
- **AutoMigrate function** - Database schema setup

**❌ What Stays Hidden**:
- **GORM models** - Internal database mapping (never exposed)
- **Middleware** - Application-specific (each app has their own)
- **Internal utilities** - Implementation details

### Database Setup Options

**Option 1: Use services.AutoMigrate() (Recommended)**

```go
db, _ := gorm.Open(...)
services.AutoMigrate(db)  // ✅ Handles all migrations
```

**Pros**:
- ✅ Simple one-line call
- ✅ Services control their schema
- ✅ Guaranteed compatibility
- ✅ No need to know internal models

**Cons**:
- ⚠️  Uses GORM AutoMigrate (not recommended for production migrations)
- ⚠️  External app doesn't control migration timing

**Option 2: Custom Migration Tool**

External apps can create their own migration tool if they need more control, but they need the schema structure (provided via AutoMigrate or documentation).

### Quick Start Guide

```bash
# Create new external app
mkdir external-app
cd external-app
go mod init github.com/myorg/external-app

# Add dependencies
go get github.com/theplant/yourapp/services
go get github.com/theplant/yourapp/api/gen/pim/v1
go get gorm.io/driver/postgres
go get gorm.io/gorm

# Create main.go with example above
# Run it!
go run main.go
```

### Common Use Cases

**1. Background Workers**

```go
import "github.com/theplant/yourapp/services"

func worker() {
    services.AutoMigrate(db)
    
    // Create service using builder pattern
    svc := services.NewProductService(db).Build()
    
    products, _, _ := svc.List(ctx, &pb.ListProductsRequest{...})
    for _, p := range products {
        // Process products
    }
}

// With optional logging for production workers
func workerWithLogging() {
    services.AutoMigrate(db)
    logger := NewWorkerLogger()
    
    // Builder pattern allows adding logger
    svc := services.NewProductService(db).
        WithLogger(logger).
        Build()
    
    products, _, _ := svc.List(ctx, &pb.ListProductsRequest{...})
    for _, p := range products {
        // Process products with logging
    }
}
```

**2. CLI Tools**

```go
import "github.com/theplant/yourapp/services"

func main() {
    services.AutoMigrate(db)
    
    // Minimal setup - database only
    svc := services.NewProductService(db).Build()
    
    product, _ := svc.Get(ctx, &pb.GetProductRequest{Id: "123"})
    fmt.Printf("Product: %s - $%.2f\n", product.Name, product.EffectivePrice)
}
```

**3. gRPC Services**

```go
import "github.com/theplant/yourapp/services"

type ProductGRPCServer struct {
    svc services.ProductService
}

func main() {
    services.AutoMigrate(db)
    logger := NewGRPCLogger()
    
    // Create service with logging for gRPC
    svc := services.NewProductService(db).
        WithLogger(logger).
        Build()
    
    grpcServer := &ProductGRPCServer{svc: svc}
    // Serve gRPC...
}
```

**4. GraphQL Resolvers**

```go
import "github.com/theplant/yourapp/services"

type Resolver struct {
    productSvc services.ProductService
}

func NewResolver(db *gorm.DB, logger Logger, cache Cache) *Resolver {
    services.AutoMigrate(db)
    
    // Builder pattern for flexible configuration
    return &Resolver{
        productSvc: services.NewProductService(db).
            WithLogger(logger).
            WithCache(cache).
            Build(),
    }
}
```

### Error Handling in External Apps

External apps should handle service errors using Go's standard error checking with `errors.Is()`:

```go
import (
    "errors"
    "github.com/theplant/yourapp/services"
)

func handleProductCreation(ctx context.Context, svc services.ProductService, req *pb.CreateProductRequest) {
    product, err := svc.Create(ctx, req)
    if err != nil {
        // Type-safe error checking with errors.Is()
        switch {
        case errors.Is(err, services.ErrDuplicateSKU):
            log.Printf("SKU already exists: %v", err)
            // Handle duplicate
        case errors.Is(err, services.ErrTemplateNotFound):
            log.Printf("Template not found: %v", err)
            // Handle missing template
        case errors.Is(err, services.ErrMissingRequired):
            log.Printf("Missing required field: %v", err)
            // Handle validation error
        case errors.Is(err, services.ErrValueOutOfRange):
            log.Printf("Value out of range: %v", err)
            // Handle range error
        case errors.Is(err, context.Canceled):
            log.Printf("Request canceled: %v", err)
            // Handle cancellation
        case errors.Is(err, context.DeadlineExceeded):
            log.Printf("Request timeout: %v", err)
            // Handle timeout
        default:
            log.Printf("Unexpected error: %v", err)
            // Handle other errors
        }
        return
    }
    
    log.Printf("✅ Created product: %s", product.Name)
}
```

**Note**: For comprehensive error handling patterns and sentinel error reference, see [Error Handling Appendix](./error-handling.md).

### Summary

**External apps get**:
- ✅ Full service business logic
- ✅ Protobuf type contracts
- ✅ Database schema (via AutoMigrate)
- ✅ Type-safe APIs

**External apps DON'T see**:
- ✅ Internal models (encapsulated)
- ✅ Internal middleware (app-specific)
- ✅ Internal utilities (hidden)

**Result**: Clean, reusable services with proper encapsulation and Go library best practices!

---

## Builder Pattern Summary

All services MUST use the builder pattern for construction. This section summarizes the complete pattern and its benefits.

### Complete Builder Pattern Example

```go
// Step 1: Define service interface (what external apps see)
type ProductService interface {
    Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error)
    Get(ctx context.Context, req *pb.GetProductRequest) (*pb.Product, error)
    // ... other methods
}

// Step 2: Define service implementation (unexported)
type productService struct {
    db            *gorm.DB  // Required
    logger        Logger    // Optional
    cache         Cache     // Optional
    createHandler func(context.Context, *pb.CreateProductRequest) (*pb.Product, error)  // For middleware
}

// Step 3: Define builder (unexported)
type productServiceBuilder struct {
    db                *gorm.DB
    logger            Logger
    cache             Cache
    createMiddlewares []CreateMiddleware  // Optional middleware
}

// Step 4: Constructor - REQUIRED parameters only
func NewProductService(db *gorm.DB) *productServiceBuilder {
    return &productServiceBuilder{db: db}
}

// Step 5: Optional configuration methods (chainable)
func (b *productServiceBuilder) WithLogger(logger Logger) *productServiceBuilder {
    b.logger = logger
    return b
}

func (b *productServiceBuilder) WithCache(cache Cache) *productServiceBuilder {
    b.cache = cache
    return b
}

func (b *productServiceBuilder) WithCreateMiddleware(mw CreateMiddleware) *productServiceBuilder {
    b.createMiddlewares = append(b.createMiddlewares, mw)
    return b
}

// Step 6: Build method - constructs final service
func (b *productServiceBuilder) Build() ProductService {
    svc := &productService{
        db:     b.db,
        logger: b.logger,
        cache:  b.cache,
    }
    
    // Build middleware chain if any
    handler := svc.coreCreate
    for i := len(b.createMiddlewares) - 1; i >= 0; i-- {
        mw := b.createMiddlewares[i]
        next := handler
        handler = func(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
            return mw(ctx, req, next)
        }
    }
    svc.createHandler = handler
    
    return svc
}
```

### Usage Patterns

```go
// Pattern 1: Minimal (tests, simple CLIs)
svc := NewProductService(db).Build()

// Pattern 2: With logging (production apps)
svc := NewProductService(db).
    WithLogger(logger).
    Build()

// Pattern 3: Full featured (production with caching)
svc := NewProductService(db).
    WithLogger(logger).
    WithCache(cache).
    Build()

// Pattern 4: With middleware (custom business rules)
svc := NewProductService(db).
    WithLogger(logger).
    WithCache(cache).
    WithCreateMiddleware(validateSKU).
    WithCreateMiddleware(auditLog).
    Build()

// Pattern 5: Conditional configuration
builder := NewProductService(db)
if config.EnableLogging {
    builder = builder.WithLogger(logger)
}
if config.EnableCache {
    builder = builder.WithCache(cache)
}
svc := builder.Build()
```

### Key Design Decisions

**1. Required vs Optional**:
- **Required**: Database connection (cannot function without it)
- **Optional**: Logger, cache, middleware (graceful degradation)

**2. Unexported Builder**:
- Builder type is unexported (`productServiceBuilder`)
- Only `Build()` returns the exported interface
- Prevents external apps from holding builder references

**3. Chainable Methods**:
- All `With*` methods return `*productServiceBuilder`
- Enables fluent API: `New().With().With().Build()`

**4. Build Once**:
- `Build()` constructs the service and middleware chain
- No performance overhead on method calls
- Immutable after construction

**5. Nil Checks**:
- Service methods check `if s.logger != nil` before using
- Allows optional dependencies to be nil
- No panics from missing optional dependencies

### Benefits Recap

| Aspect | Traditional Constructor | Builder Pattern |
|--------|------------------------|-----------------|
| Required params | Unclear | Explicit (constructor args) |
| Optional params | Must pass nil | Chainable With* methods |
| Adding new options | Breaks all callers | No breaking changes |
| Readability | `New(db, nil, nil, nil)` | `New(db).WithLogger(log).Build()` |
| Type safety | All params look equal | Required enforced by compiler |
| Flexibility | Fixed signature | Highly composable |
| External apps | Must know all params | Choose what they need |

### External App Perspective

External apps importing your services benefit from:

```go
import "github.com/theplant/yourapp/services"

// Simple start - just database
svc := services.NewProductService(db).Build()

// Grow incrementally
svc := services.NewProductService(db).
    WithLogger(theirLogger).      // Their logger implementation
    WithCache(theirCache).         // Their cache implementation
    WithCreateMiddleware(theirValidation).  // Their business rules
    Build()
```

They don't need to:
- Know about all optional dependencies
- Pass nil for unused features
- Update code when new options are added
- Match your internal structure

---

### Code Review Requirements

All pull requests MUST be reviewed against these constitutional requirements:


**Principle XI: Service Layer Architecture (Dependency Injection)**
- Reviewers MUST verify services are in public packages (not internal/)
- **Reviewers MUST verify handlers are in public packages (not internal/)**
- Reviewers MUST verify services do NOT depend on HTTP types (http.Request, http.ResponseWriter)
- **Reviewers MUST verify service method parameters are protobuf structs** (NO primitive types, NO maps)
- **Reviewers MUST verify service method return types are protobuf structs** (NOT internal models)
- Reviewers MUST reject service methods with primitive type parameters (e.g., `Get(ctx, id string)`)
- Reviewers MUST reject service methods returning multiple values (use protobuf wrapper)
- Reviewers MUST verify handlers are thin wrappers delegating to services
- Reviewers MUST verify services use builder pattern for construction:
  - Constructor accepts REQUIRED parameters only (e.g., database)
  - Optional dependencies use `With*` methods (e.g., `WithLogger`, `WithCache`)
  - Builder is unexported type, interface is exported
  - `Build()` method returns the service interface
- Reviewers MUST verify service creation uses builder pattern: `NewService(db).WithLogger(log).Build()`
- Reviewers MUST verify AutoMigrate() is updated when models change
- Reviewers MUST verify optional dependencies are checked for nil before use
