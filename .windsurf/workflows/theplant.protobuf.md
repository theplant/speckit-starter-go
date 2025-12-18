---
description: Generate Go types from Protocol Buffer definitions for gRPC or cross-language APIs.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Generate Go types from Protocol Buffer definitions for gRPC or cross-language APIs.

## Execution Steps

### Prerequisites

Install protoc and plugins:
```bash
# Install protoc (platform-specific)
# macOS: brew install protobuf
# Linux: apt install protobuf-compiler

# Install Go plugins
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

### Workflow Steps

#### 1. Define Contract
Create or update `.proto` files in `api/proto/<domain>/v1/`:

```protobuf
syntax = "proto3";

package domain.v1;

option go_package = "github.com/yourorg/yourapp/api/gen/domain/v1";

message Product {
    string id = 1;
    string name = 2;
    string sku = 3;
}

message CreateProductRequest {
    string name = 1;
    string sku = 2;
}
```

#### 2. Add go:generate Directive
```go
//go:generate protoc --go_out=. --go_opt=paths=source_relative api/proto/<domain>/v1/<domain>.proto
```

#### 3. Generate Go Code
// turbo
```bash
go generate ./...
```

Or directly:
```bash
protoc --go_out=. --go_opt=paths=source_relative api/proto/<domain>/v1/<domain>.proto
```

#### 4. Use in Code
Import generated packages, use typed structs throughout:

```go
import pb "yourapp/api/gen/domain/v1"

func (s *productService) Create(ctx context.Context, req *pb.CreateProductRequest) (*pb.Product, error) {
    // Use protobuf types
}
```

#### 5. Serialize with protojson
Use `protojson` for JSON serialization (camelCase):

```go
import "google.golang.org/protobuf/encoding/protojson"

// Marshal to JSON
data, _ := protojson.Marshal(product)

// Unmarshal from JSON
protojson.Unmarshal(body, &product)
```

#### 6. Version Control
- Never reuse deleted field numbers
- Stabilize field numbers before production use

### Testing with Protobuf

Tests MUST use `cmp.Diff()` with `protocmp.Transform()`:

```go
import (
    "github.com/google/go-cmp/cmp"
    "google.golang.org/protobuf/testing/protocmp"
)

expected := &pb.Product{
    Id:   response.Id,      // Random UUID (use response)
    Name: "Test Product",   // From request fixture
}
if diff := cmp.Diff(expected, &response, protocmp.Transform()); diff != "" {
    t.Errorf("Mismatch (-want +got):\n%s", diff)
}
```

### AI Agent Requirements

- MUST use `protojson` for JSON serialization (NOT `encoding/json`)
- MUST use `protocmp.Transform()` with `cmp.Diff()` for test comparisons
- MUST preserve field numbers and MUST NOT reuse deleted field numbers
- MUST use protobuf structs in tests (NO `map[string]interface{}`)
