---
description: Generate Protobuf from OpenAPI for projects needing both REST and gRPC (hybrid approach).
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Generate Protobuf from OpenAPI for projects needing both REST and gRPC (hybrid approach).

## Execution Steps

### Prerequisites

Install required tools:
```bash
# Install openapi2proto
go install github.com/NYTimes/openapi2proto/cmd/openapi2proto@latest

# Install protoc and Go plugin
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
```

### Workflow Steps

#### 1. Define Contract (OpenAPI is Source of Truth)
Create OpenAPI files in `api/openapi/` - this is the single source of truth:

```yaml
# api/openapi/pim.yaml
openapi: "3.0.0"
info:
  title: PIM API
  version: "1.0"
x-global-options:
  go_package: "github.com/yourorg/yourapp/api/gen/pim/v1"
components:
  schemas:
    Product:
      type: object
      x-proto-tag: 1  # Stabilize field numbers
      properties:
        id:
          type: string
          x-proto-tag: 1
        name:
          type: string
          x-proto-tag: 2
```

#### 2. Generate Proto from OpenAPI
// turbo
```bash
openapi2proto -spec api/openapi/pim.yaml -skip-rpcs -out api/proto/pim/v1/pim.proto
```

**IMPORTANT**: NEVER hand-edit generated `.proto` files

#### 3. Generate Go Code from Proto
// turbo
```bash
protoc --go_out=. --go_opt=paths=source_relative api/proto/pim/v1/pim.proto
```

#### 4. Add go:generate Directives
```go
//go:generate go run github.com/NYTimes/openapi2proto/cmd/openapi2proto -spec ../../openapi/pim.json -skip-rpcs -out ../proto/pim/v1/pim.proto
//go:generate protoc --go_out=. --go_opt=paths=source_relative ../proto/pim/v1/pim.proto
```

#### 5. Use in Code
Import generated packages, use typed structs throughout

#### 6. Update Workflow
When API changes:
1. Update OpenAPI spec (source of truth)
2. Regenerate `.proto` from OpenAPI
3. Regenerate Go code from `.proto`

// turbo
```bash
go generate ./...
```

### Field Number Stabilization

Use `x-proto-tag` in OpenAPI to stabilize protobuf field numbers:

```yaml
properties:
  id:
    type: string
    x-proto-tag: 1    # Field number 1
  name:
    type: string
    x-proto-tag: 2    # Field number 2
  # If you remove a field, NEVER reuse its x-proto-tag number
```

### Tooling Options

- Use `openapi2proto` (github.com/NYTimes/openapi2proto) to generate `.proto` from OpenAPI
- Generate messages only with `-skip-rpcs` unless adopting grpc-gateway
- Use `x-proto-tag` (in OpenAPI) to stabilize field numbers
- Use `x-global-options` (in OpenAPI) to set protobuf options such as `go_package`

### AI Agent Requirements

- MUST treat OpenAPI as the source of truth (NEVER hand-edit generated `.proto`)
- MUST regenerate `.proto` whenever OpenAPI files change
- MUST preserve `x-proto-tag` assignments and MUST NOT reuse removed field numbers
- MUST use `protojson` for JSON serialization
- MUST use `protocmp.Transform()` with `cmp.Diff()` for test comparisons
