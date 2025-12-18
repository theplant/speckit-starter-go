---
description: Setup HTTP routes using the builder pattern with optional services and middlewares.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Configure HTTP routes following Principle SERVICE_ARCHITECTURE with builder pattern.

## Routes Setup (Builder Pattern)

### Router Builder Implementation

```go
// handlers/routes.go
type Middleware func(http.Handler) http.Handler

type routerBuilder struct {
    mux              *http.ServeMux
    workflowService  services.WorkflowService
    activityService  services.ActivityService
    executionService services.ExecutionService
    db               *gorm.DB
    middlewares      []Middleware
}

func NewRouter(workflowService services.WorkflowService, db *gorm.DB) *routerBuilder {
    return &routerBuilder{workflowService: workflowService, db: db}
}

func (b *routerBuilder) WithMux(mux *http.ServeMux) *routerBuilder {
    b.mux = mux
    return b
}

func (b *routerBuilder) WithActivityService(svc services.ActivityService) *routerBuilder {
    b.activityService = svc
    return b
}

func (b *routerBuilder) WithExecutionService(svc services.ExecutionService) *routerBuilder {
    b.executionService = svc
    return b
}

func (b *routerBuilder) WithMiddlewares(mws ...Middleware) *routerBuilder {
    b.middlewares = append(b.middlewares, mws...)
    return b
}

func (b *routerBuilder) Build() http.Handler {
    mux := b.mux
    if mux == nil {
        mux = http.NewServeMux()
    }
    
    // Register routes only for non-nil services
    if b.workflowService != nil {
        h := NewWorkflowHandler(b.workflowService)
        mux.HandleFunc("POST /api/v1/workflows", h.Create)
        mux.HandleFunc("GET /api/v1/workflows/{id}", h.Get)
        mux.HandleFunc("GET /api/v1/workflows", h.List)
    }
    if b.activityService != nil {
        h := NewActivityHandler(b.activityService)
        mux.HandleFunc("GET /api/v1/activities", h.List)
    }
    // ... other optional services
    
    // Apply middlewares in order
    var handler http.Handler = mux
    for _, mw := range b.middlewares {
        handler = mw(handler)
    }
    return handler
}
```

### Default Middlewares

```go
func DefaultMiddlewares() []Middleware {
    return []Middleware{
        recoveryMiddleware,
        LoggingMiddleware(),
        requestIDMiddleware,
        CORSMiddleware(DefaultCORSConfig()),
    }
}
```

### Usage Examples

Tests (no middlewares needed):
```go
mux := handlers.NewRouter(workflowService, db).Build()
```

Production (explicit middlewares):
```go
handler := handlers.NewRouter(workflowService, db).
    WithActivityService(activityService).
    WithExecutionService(executionService).
    WithMiddlewares(handlers.DefaultMiddlewares()...).
    Build()
```

### Testing Through Root Mux (Principle SERVEHTTP_TESTING)

```go
func TestWorkflowCreate(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    
    // Use builder pattern
    service := NewWorkflowService(db).Build()
    mux := handlers.NewRouter(service, db).Build()
    
    // Test through root mux ServeHTTP
    req := httptest.NewRequest("POST", "/api/v1/workflows", body)
    rec := httptest.NewRecorder()
    
    mux.ServeHTTP(rec, req)  // ✅ Tests complete HTTP stack
    
    // Assertions...
}
```

### Why Root Mux (Not Individual Handlers)

- Calling `handler.Create(rec, req)` bypasses routing, middleware, and method matching
- Tests pass even with broken route registration (e.g., typo in path)
- Calling `mux.ServeHTTP(rec, req)` tests the complete stack and catches routing bugs

### Path Parameter Extraction (Go 1.22+)

```go
func (h *ProductHandler) Get(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")  // ✅ Use r.PathValue() for path parameters
    // ...
}
```

### AI Agent Requirements

- Routes MUST use builder pattern with optional services
- Tests MUST go through root mux ServeHTTP (NOT individual handlers)
- Path parameters MUST be extracted using `r.PathValue()` (Go 1.22+)
- Middlewares MUST be added explicitly via `WithMiddlewares()`
- All dependencies optional - register routes only for non-nil services

### See Also

- `/theplant.service` - Create services with builder pattern
- `/theplant.errors` - Implement error handling strategy
