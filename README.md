# Speckit Starter for Go Projects

A comprehensive template for Go API projects with integration testing, protobuf data structures, distributed tracing, and service architecture patterns.

## Overview

This repository provides a battle-tested constitution and templates for building production-ready Go APIs with:

- ✅ **Integration Testing First** - Real database testing, no mocking
- ✅ **Protobuf Data Structures** - Type-safe API contracts
- ✅ **Distributed Tracing** - OpenTracing instrumentation
- ✅ **Service Layer Architecture** - Reusable business logic with dependency injection
- ✅ **Comprehensive Error Handling** - Two-layer error strategy (sentinel + HTTP codes)
- ✅ **Context-Aware Operations** - Proper timeout and cancellation handling
- ✅ **Table-Driven Tests** - Comprehensive edge case coverage
- ✅ **Real Database Fixtures** - testcontainers-go for PostgreSQL

## Quick Start

### Step 1: Initialize Your Project with Spec-Kit

First, create your new project directory and initialize it with the base spec-kit:

```bash
# Create and initialize your new project
uvx --from git+https://github.com/github/spec-kit.git specify init my-awesome-api

# Navigate to your project
cd my-awesome-api
```

This creates the base `.specify` folder with the standard spec-kit structure.

### Step 2: Overlay the Go Template Constitution

Now overlay the comprehensive Go constitution and templates from this starter with a single command:

```bash
git clone --depth 1 git@github.com:theplant/speckit-starter-go.git /tmp/speckit-starter-go-$$ && cp -r /tmp/speckit-starter-go-$$/.specify . && rm -rf /tmp/speckit-starter-go-$$
```

This one-liner will:
- Clone only the latest commit (shallow clone, fast)
- Copy the `.specify` folder to your project (overwrites existing)
- Clean up the temporary clone
- Install the comprehensive Go constitution (version 1.0.0)
- Install Go-specific templates and scripts

### Step 3: Review and Customize

Review the constitution to understand the principles:

```bash
# Read the constitution
cat .specify/memory/constitution.md

# Review the templates
ls -la .specify/templates/
```

Customize the constitution and templates for your project's specific needs.

## What You Get

### Constitution (Version 1.0.0)

A comprehensive set of principles covering:

1. **Integration Testing First** - No mocking database or HTTP layers
2. **Table-Driven Test Design** - Consistent test structure
3. **Edge Case Coverage** - Validation, boundaries, errors, security
4. **Real Database Fixtures** - GORM with testcontainers-go
5. **ServeHTTP Endpoint Testing** - Full HTTP stack validation
6. **Protobuf Data Structures** - Type-safe API contracts
7. **Distributed Tracing** - OpenTracing with appropriate granularity
8. **Service Layer Architecture** - Reusable services with dependency injection
9. **Comprehensive Error Handling** - Sentinel errors + HTTP error codes
10. **Context-Aware Operations** - Proper timeout and cancellation

### Templates

- `spec-template.md` - Feature specification with user stories
- `plan-template.md` - Implementation planning
- `tasks-template.md` - Task breakdown
- `checklist-template.md` - Development checklist

### Scripts

- `copy-specify-folder.sh` - This overlay script
- `create-new-feature.sh` - Feature scaffolding
- `setup-plan.sh` - Plan generation
- `update-agent-context.sh` - Context management

## Technology Stack

- **Language**: Go 1.21+ (latest stable recommended)
- **Database**: PostgreSQL 15+ with JSONB support
- **HTTP Framework**: Standard library `net/http` with `http.ServeMux`
- **Database Access**: GORM (gorm.io/gorm)
- **Distributed Tracing**: OpenTracing
- **Protocol Buffers**: protoc with protoc-gen-go
- **Testing**: Standard library `testing` with `httptest`
- **Test Containers**: testcontainers-go with PostgreSQL module

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

Recommended structure for maximum reusability:

```
my-awesome-api/
├── .specify/              # Spec-kit configuration (this template)
│   ├── memory/
│   │   └── constitution.md
│   ├── scripts/
│   └── templates/
├── services/              # PUBLIC - reusable business logic
│   ├── product_service.go
│   ├── errors.go          # Sentinel errors
│   └── migrations.go      # AutoMigrate for external apps
├── handlers/              # PUBLIC - HTTP handlers (optional)
│   ├── product_handler.go
│   └── error_codes.go
├── api/gen/v1/            # PUBLIC - protobuf generated code
│   └── product.pb.go
├── internal/              # INTERNAL - implementation details
│   ├── models/            # GORM models (not exposed)
│   ├── middleware/
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

This template is designed to be forked and customized for your organization's needs. Feel free to:

- Modify the constitution for your tech stack
- Add organization-specific principles
- Customize templates and scripts
- Share improvements back to the community

## License

MIT License - see [LICENSE](LICENSE) file for details

## Support

For issues or questions:
- Constitution: See `.specify/memory/constitution.md`
- Spec-kit base: https://github.com/github/spec-kit
- This template: https://github.com/theplant/speckit-starter-go

---

**Version**: 1.0.0 | **Last Updated**: 2025-11-20

