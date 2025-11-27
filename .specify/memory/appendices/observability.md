# Observability Appendix

**Part of**: [Go Project Constitution](../constitution.md)  
**Covers**: Principles XII & XIV

This appendix provides detailed implementation guidelines for distributed tracing (OpenTracing) and context-aware operations.

---

## Distributed Tracing

### XII. Distributed Tracing (OpenTracing)

All API endpoints MUST be instrumented with distributed tracing at appropriate granularity:
- Each HTTP endpoint handler MUST create or continue an OpenTracing span
- Spans MUST include operation name matching the endpoint (e.g., "POST /api/products")
- Service method calls SHOULD create child spans (e.g., "ProductService.Create")
- Database operations SHOULD be traced as a single child span per transaction (NOT per SQL query)
- Individual SQL queries MUST NOT be traced (too much overhead, use database metrics instead)
- External service calls (HTTP, gRPC) MUST propagate trace context and create spans
- Error conditions MUST be logged to the active span with `span.SetTag("error", true)`
- Trace context MUST be extracted from incoming HTTP headers (e.g., `X-B3-TraceId`)
- Trace context MUST be injected into outgoing HTTP requests
- Spans MUST include relevant tags: `http.method`, `http.url`, `http.status_code`, `service.method`
- Tests MUST verify tracing instrumentation (e.g., using mock tracer or test spans)

**Rationale**: Distributed tracing provides critical observability for debugging latency issues, understanding request flows across services, identifying bottlenecks, and correlating logs across distributed systems. OpenTracing offers a vendor-neutral API compatible with Jaeger, Zipkin, and other tracing backends. Tracing at service operation level (not individual SQL queries) keeps overhead low while providing actionable insights. For SQL query analysis, use database-specific tools like `pg_stat_statements`, slow query logs, or APM database profiling.

**Example - Correct Tracing Granularity**:
```go
import (
    "net/http"
    "github.com/opentracing/opentracing-go"
    "github.com/opentracing/opentracing-go/ext"
)

// HTTP Handler - creates root span
func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    // Extract or start HTTP span
    spanCtx, _ := opentracing.GlobalTracer().Extract(
        opentracing.HTTPHeaders,
        opentracing.HTTPHeadersCarrier(r.Header),
    )
    span := opentracing.StartSpan("POST /api/products", ext.RPCServerOption(spanCtx))
    defer span.Finish()
    
    span.SetTag("http.method", r.Method)
    span.SetTag("http.url", r.URL.String())
    
    // Parse request
    var req pb.CreateProductRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        span.SetTag("error", true)
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }
    
    // Service call creates child span (one span for entire operation)
    product, err := h.service.Create(r.Context(), &req)
    if err != nil {
        span.SetTag("error", true)
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }
    
    span.SetTag("http.status_code", http.StatusCreated)
    span.SetTag("product.id", product.Id)
    
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(product)
}

// Service - creates child span
func (s *productService) Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    span, ctx := opentracing.StartSpanFromContext(ctx, "ProductService.Create")
    defer span.Finish()
    
    // Database operation - one span per transaction, not per query
    span, ctx := opentracing.StartSpanFromContext(ctx, "ProductService.Create.DB")
    defer span.Finish()
    
    // Single transaction with multiple operations
    return s.db.Transaction(func(tx *gorm.DB) error {
        product := &Product{...}
        if err := tx.Create(product).Error; err != nil {
            span.SetTag("error", true)
            return err
        }
        
        // More operations in same transaction...
        // NO individual spans for each INSERT/UPDATE/SELECT
        
        return nil
    })
}
```

**Tracing Granularity Guidelines**:
- ✅ HTTP endpoints: Root spans
- ✅ Service operations: Child spans  
- ✅ Database transactions: Child spans
- ❌ Individual SQL queries: Too granular, high overhead
- ❌ HTTP client calls within services: Should be child spans if cross-service


#### Tracing Setup

- Development environment MUST use `opentracing.NoopTracer{}` (no external dependencies)
- Tests MUST use `opentracing.NoopTracer{}` by default
- Tests that verify tracing instrumentation MAY use mock tracer to validate spans
- Production MUST configure real tracing backend (deployment-specific: Jaeger, Zipkin, Datadog, etc.)
- Production tracing configuration MUST be loaded from environment variables


---

## Context-Aware Operations

### XIV. Context-Aware Operations

All I/O and long-running operations MUST accept and respect `context.Context`:
- All service methods MUST accept `context.Context` as the first parameter
- HTTP handlers MUST extract context from `*http.Request` using `r.Context()`
- Database operations MUST use context-aware GORM methods (e.g., `db.WithContext(ctx)`)
- External HTTP calls MUST propagate context for timeout and cancellation
- gRPC calls MUST propagate context for distributed tracing and cancellation
- Long-running operations MUST check context cancellation periodically
- Context MUST be used for OpenTracing span propagation (already covered in Principle VII)
- Context cancellation MUST be respected to prevent resource leaks
- Tests MUST verify context cancellation behavior for long-running operations
- Tests MUST use `context.WithTimeout()` or `context.WithCancel()` to simulate cancellation scenarios

**Rationale**: Context provides a standard way to propagate request-scoped values (like trace IDs), handle timeouts, and enable graceful cancellation across API boundaries. This prevents resource leaks from abandoned operations, enables coordinated shutdown, supports distributed tracing, and follows Go's idiomatic patterns for concurrent programming. Without context, operations cannot be cancelled and resources may leak when clients disconnect.

**Example - Service Layer with Context**:
```go
// Service interface - context as first parameter (MANDATORY)
type ProductService interface {
    Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error)
    Get(ctx context.Context, req *pb.GetProductRequest) (*pb.Product, error)
    List(ctx context.Context, req *pb.ListProductsRequest) (*pb.ListProductsResponse, error)
    Update(ctx context.Context, req *pb.UpdateProductRequest) (*pb.Product, error)
    Delete(ctx context.Context, req *pb.DeleteProductRequest) (*pb.DeleteProductResponse, error)
}

// Service implementation respects context
func (s *productService) Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    // Start tracing span from context
    span, ctx := opentracing.StartSpanFromContext(ctx, "ProductService.Create")
    defer span.Finish()
    
    // Validate business rules
    if req.Name == "" {
        return nil, errors.New("product name required")
    }
    
    // Create entity
    product := &Product{
        Name:        req.Name,
        SKU:         req.Sku,
        Description: req.Description,
    }
    
    // Use context-aware database operations (MANDATORY)
    tx := s.db.WithContext(ctx).Begin()
    defer tx.Rollback()
    
    // Check context cancellation before expensive operations
    select {
    case <-ctx.Done():
        return nil, ctx.Err() // Return context error (timeout or cancellation)
    default:
        // Continue with operation
    }
    
    if err := tx.Create(product).Error; err != nil {
        s.logger.Error("failed to create product", err)
        return nil, err
    }
    
    if err := tx.Commit().Error; err != nil {
        return nil, err
    }
    
    return toProto(product), nil
}
```

**HTTP Handler - Extract Context**:
```go
// HTTP handler extracts context from request
func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    // Extract context from HTTP request (MANDATORY)
    ctx := r.Context()
    
    // Create tracing span with context
    span, ctx := opentracing.StartSpanFromContext(ctx, "POST /api/products")
    defer span.Finish()
    
    // Parse request
    var req pb.CreateProductRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request", 400)
        return
    }
    
    // Pass context to service (context propagation)
    product, err := h.service.Create(ctx, &req)
    if err != nil {
        // Check if error was due to context cancellation
        if err == context.Canceled {
            http.Error(w, "Request cancelled", 499) // Client closed connection
            return
        }
        if err == context.DeadlineExceeded {
            http.Error(w, "Request timeout", 504) // Gateway timeout
            return
        }
        
        span.SetTag("error", true)
        http.Error(w, err.Error(), 500)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(product)
}
```

**External HTTP Calls with Context**:
```go
// Making HTTP calls to external services with context propagation
func (s *productService) FetchExternalData(ctx context.Context, url string) (*Data, error) {
    // Create HTTP request with context (MANDATORY)
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    
    // Inject tracing headers
    carrier := opentracing.HTTPHeadersCarrier(req.Header)
    if err := opentracing.GlobalTracer().Inject(
        opentracing.SpanFromContext(ctx).Context(),
        opentracing.HTTPHeaders,
        carrier,
    ); err != nil {
        return nil, err
    }
    
    // Make request - will respect context timeout/cancellation
    resp, err := s.httpClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    var data Data
    if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
        return nil, err
    }
    
    return &data, nil
}
```

**Testing Context Cancellation**:
```go
func TestProductService_Create_ContextCancellation(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    // Use builder pattern for service construction
    service := NewProductService(db).
        WithLogger(logger).
        WithCache(cache).
        Build()
    
    // Create context with immediate cancellation
    ctx, cancel := context.WithCancel(context.Background())
    cancel() // Cancel immediately
    
    req := &pb.CreateProductRequest{
        Name: "Test Product",
        Sku:  "TEST-001",
    }
    
    // Service should respect cancellation
    product, err := service.Create(ctx, req)
    
    // Verify context cancellation was respected
    if err != context.Canceled {
        t.Errorf("Expected context.Canceled error, got: %v", err)
    }
    if product != nil {
        t.Error("Expected nil product when context cancelled")
    }
    
    // Verify no data was committed to database
    var count int64
    db.Model(&Product{}).Count(&count)
    if count != 0 {
        t.Error("Expected no products in database after cancellation")
    }
}

func TestProductService_Create_ContextTimeout(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    // Use builder pattern for service construction
    service := NewProductService(db).
        WithLogger(logger).
        WithCache(cache).
        Build()
    
    // Create context with very short timeout
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Nanosecond)
    defer cancel()
    
    time.Sleep(10 * time.Millisecond) // Ensure timeout expires
    
    req := &pb.CreateProductRequest{
        Name: "Test Product",
        Sku:  "TEST-001",
    }
    
    // Service should respect timeout
    product, err := service.Create(ctx, req)
    
    // Verify timeout was respected
    if err != context.DeadlineExceeded {
        t.Errorf("Expected context.DeadlineExceeded error, got: %v", err)
    }
    if product != nil {
        t.Error("Expected nil product when context timeout")
    }
}
```

**Long-Running Operations**:
```go
// For operations that process many items, check context periodically
func (s *productService) BulkUpdate(ctx context.Context, updates []*pb.ProductUpdate) error {
    tx := s.db.WithContext(ctx).Begin()
    defer tx.Rollback()
    
    for i, update := range updates {
        // Check context cancellation every N iterations
        if i%100 == 0 {
            select {
            case <-ctx.Done():
                return ctx.Err() // Stop processing if cancelled
            default:
                // Continue processing
            }
        }
        
        // Perform update
        if err := tx.Model(&Product{}).
            Where("id = ?", update.Id).
            Updates(update).Error; err != nil {
            return err
        }
    }
    
    return tx.Commit().Error
}
```


### Code Review Requirements

All pull requests MUST be reviewed against these constitutional requirements:


**Principle XII: Distributed Tracing (OpenTracing)**
- Reviewers MUST verify OpenTracing spans are created for new endpoints
- Reviewers MUST verify spans include required tags (http.method, http.url, http.status_code)
- Reviewers MUST verify error spans are tagged with error=true

**Principle XIV: Context-Aware Operations**
- Reviewers MUST verify all service methods accept `context.Context` as first parameter
- Reviewers MUST verify HTTP handlers extract context from `r.Context()`
- Reviewers MUST verify database operations use `db.WithContext(ctx)`
- Reviewers MUST verify external HTTP calls use `http.NewRequestWithContext(ctx, ...)`
- Reviewers MUST verify context is propagated through all layers (HTTP → Service → Repository)
- Reviewers MUST verify context cancellation checks in Create/long-running operations
- Tests MUST verify context cancellation behavior where applicable
