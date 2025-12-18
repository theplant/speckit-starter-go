---
description: Implement comprehensive error handling with sentinel errors and automatic HTTP mapping.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Implement comprehensive error handling with sentinel errors and automatic HTTP mapping (Principle ERROR_HANDLING).

## Two-Layer Error Handling Strategy

### Service Layer: Sentinel Errors

Define sentinel errors in `services/errors.go`:

```go
package services

import "errors"

var (
    ErrProductNotFound = errors.New("product not found")
    ErrDuplicateSKU    = errors.New("SKU already exists")
    ErrMissingRequired = errors.New("required field missing")
)
```

### HTTP Layer: Error Code Singleton

Define error codes in `handlers/error_codes.go`:

```go
package handlers

type ErrorCode struct {
    Code       string
    Message    string
    HTTPStatus int
    ServiceErr error  // Maps to service sentinel error
}

var Errors = struct {
    MissingRequired  ErrorCode
    ProductNotFound  ErrorCode
    DuplicateSKU     ErrorCode
    RequestCancelled ErrorCode
    RequestTimeout   ErrorCode
    InternalError    ErrorCode
}{
    MissingRequired:  ErrorCode{"MISSING_REQUIRED", "Required field missing", 400, services.ErrMissingRequired},
    ProductNotFound:  ErrorCode{"PRODUCT_NOT_FOUND", "Product not found", 404, services.ErrProductNotFound},
    DuplicateSKU:     ErrorCode{"DUPLICATE_SKU", "SKU already exists", 409, services.ErrDuplicateSKU},
    RequestCancelled: ErrorCode{"REQUEST_CANCELLED", "Request cancelled", 499, context.Canceled},
    RequestTimeout:   ErrorCode{"REQUEST_TIMEOUT", "Request timeout", 504, context.DeadlineExceeded},
    InternalError:    ErrorCode{"INTERNAL_ERROR", "Internal server error", 500, nil},
}
```

### Service: Wrap Errors with Context

```go
func (s *productService) Get(ctx context.Context, req *pb.GetProductRequest) (*pb.Product, error) {
    // Validate in service (handler does NOT validate)
    if req.Id == "" {
        return nil, fmt.Errorf("product id: %w", ErrMissingRequired)
    }
    
    var product Product
    if err := s.db.WithContext(ctx).First(&product, "id = ?", req.Id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("get product %s: %w", req.Id, ErrProductNotFound)
        }
        return nil, fmt.Errorf("query product %s: %w", req.Id, err)
    }
    return toProto(&product), nil
}
```

### Handler: Automatic Error Mapping

```go
func HandleServiceError(w http.ResponseWriter, err error) {
    // Check context errors first (Principle CONTEXT_AWARE)
    if errors.Is(err, context.Canceled) {
        RespondWithError(w, Errors.RequestCanceled, err)
        return
    }
    if errors.Is(err, context.DeadlineExceeded) {
        RespondWithError(w, Errors.RequestTimeout, err)
        return
    }
    
    // Check service error mapping
    for _, errCode := range AllErrors() {
        if errCode.ServiceErr != nil && errors.Is(err, errCode.ServiceErr) {
            RespondWithError(w, errCode, err)  // Pass original error for details
            return
        }
    }
    RespondWithError(w, Errors.InternalError, err)
}
```

### Environment-Aware Error Details

```go
// Configuration
var defaultErrorConfig = &errorResponseConfig{hideErrorDetails: false}

func SetHideErrorDetails(hide bool) {
    defaultErrorConfig.hideErrorDetails = hide
}

// Response helper
func RespondWithError(w http.ResponseWriter, errCode ErrorCode, err error) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(errCode.HTTPStatus)
    
    errResp := &pb.ErrorResponse{
        Code:    errCode.Code,
        Message: errCode.Message,
    }
    
    // Include full error chain unless hidden
    if !defaultErrorConfig.hideErrorDetails && err != nil {
        errResp.Details = err.Error()
    }
    
    data, _ := protojson.Marshal(errResp)
    w.Write(data)
}
```

### Startup Configuration

```go
// cmd/api/main.go
func main() {
    if os.Getenv("HIDE_ERROR_DETAILS") == "true" {
        handlers.SetHideErrorDetails(true)
    }
    // ... rest of setup
}
```

### Example Responses

Development (default):
```json
{
    "code": "PRODUCT_NOT_FOUND",
    "message": "Product not found",
    "details": "get product abc-123: product not found"
}
```

Production (`HIDE_ERROR_DETAILS=true`):
```json
{
    "code": "PRODUCT_NOT_FOUND",
    "message": "Product not found"
}
```

### Testing Errors

```go
func TestProductAPI_Create_DuplicateSKU(t *testing.T) {
    // ... setup ...
    
    // Use error code definitions (NOT literal strings)
    var errResp pb.ErrorResponse
    protojson.Unmarshal(respBody, &errResp)
    
    if errResp.Code != Errors.DuplicateSKU.Code {
        t.Errorf("Expected %s, got %s", Errors.DuplicateSKU.Code, errResp.Code)
    }
}
```

### AI Agent Requirements

- MUST read the `details` field when debugging API errors
- MUST NOT assume `details` is always present (production hides it)
- SHOULD suggest setting `HIDE_ERROR_DETAILS=true` for production
- Tests MUST use `ErrorCode` definitions (NOT literal strings)
- ALL errors (sentinel + HTTP codes) MUST have test cases

### See Also

- `/theplant.service` - Create services with builder pattern
- `/theplant.routes` - Setup HTTP routing for services
