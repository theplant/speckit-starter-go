---
description: Generate Go types and server interfaces from OpenAPI specifications using oapi-codegen.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Generate Go types and server interfaces directly from OpenAPI specifications using oapi-codegen.

## Execution Steps

### Prerequisites

Install oapi-codegen if not available:
```bash
go install github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@latest
```

### Workflow Steps

#### 1. Define Contract
Create or update OpenAPI files in `api/openapi/<domain>.yaml` (YAML preferred) or `.json`

#### 2. Configure oapi-codegen
Create configuration file at `api/gen/<domain>/oapi-codegen.yaml`:

```yaml
package: <domain>
generate:
  std-http-server: true    # Go 1.22+ net/http server
  strict-server: true      # Type-safe request/response handling
  models: true             # Generate model types
  embedded-spec: true      # Embed spec for validation
output: api.gen.go
```

#### 3. Add go:generate Directive
Add to a Go file in the package:

```go
//go:generate go run github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen --config=oapi-codegen.yaml ../../openapi/<domain>.yaml
```

#### 4. Generate Go Code
// turbo
```bash
go generate ./...
```

#### 5. Implement Interface
Implement the generated `StrictServerInterface`:

```go
type ProductServer struct {
    db *gorm.DB
}

func (s *ProductServer) CreateProduct(ctx context.Context, request CreateProductRequestObject) (CreateProductResponseObject, error) {
    // Business logic here - request.Body contains typed CreateProductRequest
    product := &models.Product{Name: request.Body.Name}
    if err := s.db.WithContext(ctx).Create(product).Error; err != nil {
        return nil, err
    }
    return CreateProduct201JSONResponse{Id: product.ID, Name: product.Name}, nil
}
```

#### 6. Use in Code
Import generated packages, use typed structs throughout

#### 7. Update
Regenerate code whenever OpenAPI spec changes:
// turbo
```bash
go generate ./...
```

### Key Points

- **CRITICAL**: Services MUST NOT define their own interfaces - implement `StrictServerInterface` directly
- Business logic handlers implement `StrictServerInterface` - this IS the service layer
- Tests MUST use generated structs (NO `map[string]interface{}`)
- Tests MUST compare structs using `cmp.Diff()` (NO `==`, `reflect.DeepEqual`, or individual field checks)

### AI Agent Requirements

- MUST check if OpenAPI spec exists in `api/openapi/` before generating code
- MUST install `oapi-codegen` if not available
- MUST regenerate code whenever OpenAPI files change
- MUST implement the generated `StrictServerInterface` for type-safe handlers
- MUST use generated request/response types (NO manual struct definitions)
