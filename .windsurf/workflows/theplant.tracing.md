---
description: Implement distributed tracing with OpenTracing following constitution Principle DISTRIBUTED_TRACING.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Implement distributed tracing with OpenTracing following Principle DISTRIBUTED_TRACING.

## Execution Steps

First, create a task plan using `update_plan`:

```
Call update_plan with:
- explanation: "Implement distributed tracing for <service>"
- plan: [
    {"step": "Add OpenTracing dependency", "status": "pending"},
    {"step": "Add spans to HTTP handlers", "status": "pending"},
    {"step": "Add child spans to service methods", "status": "pending"},
    {"step": "Configure context propagation", "status": "pending"},
    {"step": "Setup noop tracer for tests", "status": "pending"},
    {"step": "Configure production tracer (Jaeger)", "status": "pending"}
  ]
```

Then execute each step, updating status to `in_progress` before starting and `completed` after finishing.

## Distributed Tracing

### Requirements

- HTTP endpoints MUST create OpenTracing spans with operation name
- Service methods SHOULD create child spans
- Database operations: ONE span per transaction (NOT per SQL query)
- External calls MUST propagate trace context
- Errors MUST set `span.SetTag("error", true)`
- Spans MUST include tags: `http.method`, `http.url`, `http.status_code`

### HTTP Handler with Tracing

```go
import "github.com/opentracing/opentracing-go"

func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    span := opentracing.StartSpan("POST /api/products")
    defer span.Finish()
    span.SetTag("http.method", r.Method)
    
    var req pb.CreateProductRequest
    if err := DecodeProtoJSON(r, &req); err != nil {
        span.SetTag("error", true)
        RespondWithError(w, Errors.InvalidRequest, err)
        return
    }
    
    product, err := h.service.Create(r.Context(), &req)
    if err != nil {
        span.SetTag("error", true)
        HandleServiceError(w, err)
        return
    }
    
    span.SetTag("http.status_code", http.StatusCreated)
    RespondWithProto(w, http.StatusCreated, product)
}
```

### Service with Child Span

```go
func (s *productService) Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    span, ctx := opentracing.StartSpanFromContext(ctx, "ProductService.Create")
    defer span.Finish()
    
    product := &Product{Name: req.Name, SKU: req.Sku}
    if err := s.db.WithContext(ctx).Create(product).Error; err != nil {
        span.SetTag("error", true)
        return nil, err
    }
    
    return toProto(product), nil
}
```

### Tracing Granularity

**DO trace**:
- HTTP endpoints
- Service operations
- Database transactions (one span per transaction)
- External HTTP/gRPC calls

**DON'T trace**:
- Individual SQL queries (too much overhead)
- Internal helper functions
- Simple data transformations

### Setup

Development/Tests:
```go
import "github.com/opentracing/opentracing-go"

// Use noop tracer
opentracing.SetGlobalTracer(opentracing.NoopTracer{})
```

Production (configure from environment):
```go
// Example with Jaeger
import (
    "github.com/uber/jaeger-client-go"
    "github.com/uber/jaeger-client-go/config"
)

func initTracer(serviceName string) (opentracing.Tracer, io.Closer, error) {
    cfg := config.Configuration{
        ServiceName: serviceName,
        Sampler: &config.SamplerConfig{
            Type:  jaeger.SamplerTypeConst,
            Param: 1,
        },
        Reporter: &config.ReporterConfig{
            LogSpans: true,
        },
    }
    return cfg.NewTracer()
}
```

### Context Propagation

Always propagate context through the call chain:

```go
// HTTP Handler
func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()  // Extract context
    result, err := h.service.Get(ctx, req)  // Propagate
}

// Service
func (s *service) Get(ctx context.Context, req *pb.Request) (*pb.Response, error) {
    span, ctx := opentracing.StartSpanFromContext(ctx, "Service.Get")
    defer span.Finish()
    
    // Use ctx for database operations
    s.db.WithContext(ctx).First(&model, id)
}
```

### AI Agent Requirements

- HTTP endpoints MUST create spans with operation name
- Service methods SHOULD create child spans
- Database: ONE span per transaction (NOT per query)
- Errors MUST set `span.SetTag("error", true)`
- Use `opentracing.NoopTracer{}` for tests
- Configure real tracer from environment in production
