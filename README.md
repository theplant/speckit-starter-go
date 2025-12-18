# Speckit Starter for Go Backend APIs

A comprehensive template for Go backend API projects with integration testing, protobuf data structures, distributed tracing, and service architecture patterns.

## Overview

This repository provides a battle-tested constitution, templates, and AI workflows for building production-ready Go backend APIs with:

- ✅ **Integration Testing First** - Real database testing, no mocking
- ✅ **Protobuf Data Structures** - Type-safe API contracts
- ✅ **Distributed Tracing** - OpenTracing instrumentation
- ✅ **Service Layer Architecture** - Reusable business logic with dependency injection
- ✅ **Comprehensive Error Handling** - Two-layer error strategy (sentinel + HTTP codes)
- ✅ **Context-Aware Operations** - Proper timeout and cancellation handling
- ✅ **Table-Driven Tests** - Comprehensive edge case coverage
- ✅ **Real Database Fixtures** - testcontainers-go for PostgreSQL
- ✅ **AI Workflows** - Windsurf workflows for TDD, debugging, and code generation

## Quick Start

Run this one-liner in your existing Go project directory:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/theplant/speckit-starter-go/HEAD/install.sh)"
```

This will:
- Install `uv` via Homebrew if not available
- Initialize spec-kit with Windsurf AI support
- Install the Go constitution, templates, and AI workflows
- Set up `.specify/` and `.windsurf/workflows/` folders

### What Gets Installed

After running the installer:

```
your-project/
├── .specify/
│   ├── memory/
│   │   └── constitution.md    # Go backend principles
│   ├── templates/             # Spec, plan, task templates
│   └── scripts/               # Helper scripts
└── .windsurf/
    └── workflows/
        ├── theplant.tdd.md              # Test-First Development
        ├── theplant.bugfix.md           # Reproduction-first debugging
        ├── theplant.service.md          # Service layer architecture
        ├── theplant.routes.md           # HTTP routing setup
        ├── theplant.errors.md           # Error handling strategy
        ├── theplant.coverage.md         # Test coverage analysis
        ├── theplant.testdb.md           # Test database setup
        ├── theplant.openapi.md          # OpenAPI code generation
        ├── theplant.protobuf.md         # Protobuf code generation
        ├── theplant.tracing.md          # Distributed tracing
        ├── theplant.system-exploration.md  # Code path tracing
        └── theplant.root-cause-tracing.md  # Debugging discipline
```

### Review and Customize

```bash
# Read the constitution
cat .specify/memory/constitution.md

# List available AI workflows
ls .windsurf/workflows/theplant.*.md
```

## What You Get

### Constitution Principles

A comprehensive set of principles for backend API development:

| Principle | Description |
|-----------|-------------|
| `INTEGRATION_TESTING` | No mocking database or HTTP layers |
| `TABLE_DRIVEN` | Consistent test structure with table-driven design |
| `EDGE_CASE_COVERAGE` | Validation, boundaries, errors, security |
| `REAL_FIXTURES` | GORM with testcontainers-go |
| `SERVEHTTP_TESTING` | Full HTTP stack validation through root mux |
| `SCHEMA_FIRST` | Type-safe API contracts with protobuf |
| `DISTRIBUTED_TRACING` | OpenTracing with appropriate granularity |
| `SERVICE_ARCHITECTURE` | Reusable services with dependency injection |
| `ERROR_HANDLING` | Sentinel errors + HTTP error codes |
| `CONTEXT_AWARE` | Proper timeout and cancellation |
| `CONTINUOUS_TESTING` | Run tests after every code change |
| `ROOT_CAUSE` | Trace problems to source, fix root cause |
| `ACCEPTANCE_COVERAGE` | All errors must have test cases |

### AI Workflows

Use these workflows in Windsurf with `/theplant.<workflow>` commands:

| Workflow | Purpose |
|----------|--------|
| `/theplant.tdd` | Test-First Development for new features |
| `/theplant.bugfix` | Reproduction-first debugging |
| `/theplant.service` | Create services with builder pattern |
| `/theplant.routes` | Setup HTTP routing |
| `/theplant.errors` | Implement error handling strategy |
| `/theplant.coverage` | Analyze and improve test coverage |
| `/theplant.testdb` | Setup test database with testcontainers |
| `/theplant.openapi` | Generate Go code from OpenAPI specs |
| `/theplant.protobuf` | Generate Go code from protobuf |
| `/theplant.hybrid-api` | OpenAPI → Protobuf for REST+gRPC |
| `/theplant.tracing` | Implement distributed tracing |
| `/theplant.system-exploration` | Trace code paths before writing tests |
| `/theplant.root-cause-tracing` | Debug with root cause discipline |

### Templates

- `spec-template.md` - Feature specification with user stories
- `plan-template.md` - Implementation planning
- `tasks-template.md` - Task breakdown
- `checklist-template.md` - Development checklist

## Technology Stack

- **Language**: Go 1.22+ (for `r.PathValue()` support)
- **Database**: PostgreSQL 15+ with JSONB support
- **HTTP Framework**: Standard library `net/http` with `http.ServeMux`
- **Database Access**: GORM (gorm.io/gorm)
- **Distributed Tracing**: OpenTracing
- **Protocol Buffers**: protoc with protoc-gen-go
- **Testing**: Standard library `testing` with `httptest`
- **Test Containers**: testcontainers-go with PostgreSQL module
- **JSON Serialization**: `protojson` for protobuf types

## Development Workflow

### 1. Define API Contract

Create `.proto` files for your API:

```protobuf
// api/product.proto
syntax = "proto3";

package product.v1;

message ProductCreateRequest {
  string name = 1;
  string sku = 2;
}

message Product {
  string id = 1;
  string name = 2;
  string sku = 3;
}
```

### 2. Write Tests First (TDD)

Create table-driven integration tests:

```go
func TestProductCreate(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products")
    
    testCases := []struct {
        name           string
        request        *pb.ProductCreateRequest
        expectedStatus int
    }{
        {
            name: "valid product",
            request: &pb.ProductCreateRequest{
                Name: "Test Product",
                Sku:  "TEST-001",
            },
            expectedStatus: http.StatusCreated,
        },
        // More test cases...
    }
    
    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            // Test implementation
        })
    }
}
```

### 3. Implement Service Layer

Create reusable services with dependency injection:

```go
type ProductService interface {
    Create(ctx context.Context, req *pb.ProductCreateRequest) (*pb.Product, error)
}

type productService struct {
    db *gorm.DB
}

func NewProductService(db *gorm.DB) ProductService {
    return &productService{db: db}
}
```

### 4. Implement HTTP Handlers

Thin wrappers that delegate to services:

```go
type ProductHandler struct {
    service ProductService
}

func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    var req pb.ProductCreateRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        RespondWithError(w, Errors.InvalidRequest)
        return
    }
    
    product, err := h.service.Create(ctx, &req)
    if err != nil {
        HandleServiceError(w, err)
        return
    }
    
    json.NewEncoder(w).Encode(product)
}
```

## Key Principles

### Integration Testing

Tests MUST use real database connections (no mocking):

```go
func setupTestDB(t *testing.T) (*gorm.DB, func()) {
    ctx := context.Background()
    
    pgContainer, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:15-alpine"),
        postgres.WithDatabase("testdb"),
        // ... configuration
    )
    // ... setup code
}
```

### Protobuf Assertions

MUST use `cmp.Diff()` with `protocmp.Transform()`:

```go
// CORRECT
if diff := cmp.Diff(expected, actual, protocmp.Transform()); diff != "" {
    t.Errorf("Mismatch (-want +got):\n%s", diff)
}

// WRONG - individual field comparison
if actual.Name != expected.Name {
    t.Error("name mismatch")
}
```

### Error Handling

Two-layer strategy with automatic mapping:

```go
// Service layer - sentinel errors
var (
    ErrProductNotFound = errors.New("product not found")
    ErrDuplicateSKU    = errors.New("SKU already exists")
)

// HTTP layer - error codes with service mapping
var Errors = struct {
    ProductNotFound ErrorCode
}{
    ProductNotFound: ErrorCode{
        Code:       "PRODUCT_NOT_FOUND",
        Message:    "Product not found",
        HTTPStatus: http.StatusNotFound,
        ServiceErr: services.ErrProductNotFound, // Automatic mapping
    },
}
```

## Project Structure

Recommended structure for Go backend API projects (maximum reusability):

```
my-awesome-api/
├── .specify/              # Spec-kit configuration
│   ├── memory/
│   │   └── constitution.md
│   ├── scripts/
│   └── templates/
├── .windsurf/
│   └── workflows/         # AI workflows for development
│       └── theplant.*.md
├── api/
│   ├── openapi/           # OpenAPI specifications
│   ├── proto/             # Protobuf source files
│   └── gen/               # Generated Go code
├── services/              # PUBLIC - reusable business logic
│   ├── product_service.go
│   └── errors.go          # Sentinel errors
├── handlers/              # PUBLIC - HTTP handlers
│   ├── product_handler.go
│   ├── routes.go          # Router builder
│   └── error_codes.go     # HTTP error mapping
├── internal/              # INTERNAL - implementation details
│   ├── models/            # GORM models (not exposed)
│   └── config/
└── tests/
    └── integration/
        └── product_test.go
```

## Testing Requirements

### Mandatory Test Coverage

Every feature MUST test:

- ✅ Happy path scenarios
- ✅ Input validation (empty, nil, invalid, SQL injection, XSS)
- ✅ Boundary conditions (zero, negative, max values)
- ✅ Authentication & authorization
- ✅ Data state (404, conflicts, concurrency)
- ✅ Database errors (constraints, transactions)
- ✅ HTTP specifics (wrong methods, headers, malformed JSON)
- ✅ Context cancellation and timeouts
- ✅ **ALL sentinel errors** (services/errors.go)
- ✅ **ALL HTTP error codes** (handlers/error_codes.go)

### Test Isolation

Use database truncation for cleanup:

```go
func TestProduct(t *testing.T) {
    db, cleanup := setupTestDB(t)
    defer cleanup()
    defer truncateTables(db, "products", "orders")
    
    // Test implementation
}
```

## Contributing

This template is designed to be forked and customized for your organization's backend API needs. Feel free to:

- Modify the constitution for your specific backend stack
- Add organization-specific API principles
- Customize templates and scripts
- Share improvements back to the community

**Note**: This template is specifically for Go backend APIs. If you need frontend or mobile app templates, consider creating separate specialized templates.

## License

MIT License - see [LICENSE](LICENSE) file for details

## Support

For issues or questions:
- Constitution: See `.specify/memory/constitution.md`
- Spec-kit base: https://github.com/github/spec-kit
- This template: https://github.com/theplant/speckit-starter-go

