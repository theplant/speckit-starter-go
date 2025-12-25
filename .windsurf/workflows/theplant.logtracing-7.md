---
description: Implement distributed tracing using logtracing from theplant/appkit for Go services.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Automatically integrate `github.com/theplant/appkit/logtracing` into the project:

1. Check and add dependencies, configuration, and logger setup
2. Add tracing to Services and Handlers with **meaningful business context**

## Execution Steps

### Phase 1: Project Analysis

1. Read `go.mod` to get module name and check dependencies
2. Find main entry point (`cmd/*/main.go` or `main.go`)
3. Find config files (`deploy/`, `config/`, or root)
4. **Analyze project structure to identify**:
   - Service layer directory (look for files with service structs, business logic, `ctx context.Context` parameters)
   - Handler layer directory (look for files with HTTP handlers, `http.ResponseWriter`, `*http.Request` parameters)
   - App configuration directory (look for Config struct, providers)

### Phase 2: Dependencies & Configuration

1. **Check go.mod** for:

   - `github.com/theplant/appkit/logtracing`
   - `github.com/qor5/x/v3/slogx`
   - If missing, run `go get` to add

2. **Check Config struct** for `Logging *slogx.Config`:

   - If missing, add field with tag `confx:"logging"`

3. **Check YAML config** for `logging:` section:

   - If missing, add `logging: { level: "info" }`

4. **Check SetupLogger provider**:

   - If missing, create in app config directory:

   ```go
   var SetupLogger = []any{
       func(conf *Config) *slogx.Config { return conf.Logging },
       slogx.SetupDefaultLogger,
   }
   ```

5. **Check main.go** for logger init before other components

### Phase 3: Service Layer Tracing

1. **Find service layer directory** identified in Phase 1
2. For each function with `ctx context.Context` parameter WITHOUT `logtracing.StartSpan`:
   - **Analyze function logic and parameters** to determine what to record
   - Add tracing with **meaningful span content** (see Span Content Rules)
   - Ensure error return is named `xerr`
   - Add import if missing

### Phase 4: Handler Layer Tracing

1. **Find handler layer directory** identified in Phase 1
2. For each HTTP handler WITHOUT `logtracing.StartSpan`:
   - **Analyze handler logic** to determine what to record
   - Add tracing with **meaningful span content** (see Span Content Rules)
   - Add import if missing

### Phase 5: Verification

1. Run `go build` to verify
2. Run `go mod tidy`
3. Report: files modified, functions traced, manual actions needed

## Execution Rules

- Process one file at a time
- Always add imports when adding logtracing
- Preserve existing code logic
- Skip functions with existing tracing
- Skip `*_test.go` and generated files

---

## Span Content Rules (CRITICAL)

**DO NOT just record the function name. Analyze the function and record meaningful business context.**

### What to Record

| Function Type         | What to Record                                  |
| --------------------- | ----------------------------------------------- |
| CRUD operations       | Entity ID, entity type, operation result        |
| List/Query operations | Query parameters, result count, pagination info |
| Create operations     | Created entity ID, key input fields             |
| Update operations     | Entity ID, fields being updated                 |
| Delete operations     | Entity ID, deletion result                      |
| Batch operations      | Batch size, success/failure counts              |
| Cron jobs             | Job name, execution time, items processed       |
| External API calls    | API name, request params, response status       |
| Authentication        | User ID, auth method, success/failure           |

### How to Analyze

1. **Look at function parameters** - Record IDs, filters, pagination params
2. **Look at function return values** - Record counts, created IDs, status
3. **Look at function logic** - Record key decision points, loop counts
4. **Look at error handling** - Record error context when failures occur

### Examples

**❌ Wrong - No business context:**

```go
func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    ctx, _ = logtracing.StartSpan(ctx, "handler.CreateOrder")
    defer func() {
        logtracing.EndSpan(ctx, nil)
    }()
    // ...
}
```

**✅ Correct - With business context:**

```go
func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    ctx, _ = logtracing.StartSpan(ctx, "handler.CreateOrder")
    defer func() {
        logtracing.EndSpan(ctx, nil)
    }()

    // After parsing request
    logtracing.AppendSpanKVs(ctx, "user_id", req.UserID, "items_count", len(req.Items))

    // After creating order
    logtracing.AppendSpanKVs(ctx, "order_id", order.ID, "total_amount", order.TotalAmount)
    // ...
}
```

**✅ Correct - Cron job:**

```go
func (s *Service) ProcessDailyReport(ctx context.Context) (xerr error) {
    ctx, _ = logtracing.StartSpan(ctx, "ReportService.ProcessDailyReport")
    defer func() {
        logtracing.EndSpan(ctx, xerr)
    }()

    startTime := time.Now()
    // ... processing logic ...

    logtracing.AppendSpanKVs(ctx,
        "job_name", "daily_report",
        "execution_date", time.Now().Format("2006-01-02"),
        "records_processed", processedCount,
        "duration_ms", time.Since(startTime).Milliseconds(),
    )
    return nil
}
```

**✅ Correct - List operation:**

```go
func (s *Service) ListItems(ctx context.Context, params *ListParams) (result *ListResult, xerr error) {
    ctx, _ = logtracing.StartSpan(ctx, "ItemService.ListItems")
    defer func() {
        count := 0
        if result != nil {
            count = len(result.Items)
        }
        logtracing.AppendSpanKVs(ctx,
            "filter_status", params.Status,
            "limit", params.Limit,
            "offset", params.Offset,
            "result_count", count,
            "has_more", result != nil && result.HasMore,
        )
        logtracing.EndSpan(ctx, xerr)
    }()
    // ...
}
```

---

## Critical Rules

| Rule                 | Description                                                      |
| -------------------- | ---------------------------------------------------------------- |
| Named return         | **MUST** use `xerr` for error capture in defer                   |
| Pass new ctx         | **MUST** pass ctx returned by `StartSpan` to children            |
| Defer EndSpan        | **MUST** call `EndSpan` in defer                                 |
| Setup slog first     | **MUST** setup slog before using logtracing                      |
| Span naming          | Use `PackageName.FunctionName` or `TypeName.MethodName`          |
| Key naming           | **MUST** use snake_case: `user_id`, not `userId`                 |
| **Business context** | **MUST** record meaningful business data, not just function name |

### Common Mistakes

```go
// ❌ Wrong: Ignoring returned ctx
_, _ = logtracing.StartSpan(ctx, "MySpan")

// ✅ Correct
ctx, _ = logtracing.StartSpan(ctx, "MySpan")
```

```go
// ❌ Wrong: No business context recorded
ctx, _ = logtracing.StartSpan(ctx, "handler.CreateOrder")
defer func() { logtracing.EndSpan(ctx, nil) }()

// ✅ Correct: Record IDs, counts, key data
ctx, _ = logtracing.StartSpan(ctx, "handler.CreateOrder")
defer func() { logtracing.EndSpan(ctx, nil) }()
logtracing.AppendSpanKVs(ctx, "order_id", orderID, "user_id", userID)
```

---

## Code Templates

### Pattern A: Basic Tracing with Context

```go
func (s *Service) GetByID(ctx context.Context, id string) (result *Entity, xerr error) {
    ctx, _ = logtracing.StartSpan(ctx, "ServiceName.GetByID")
    defer func() {
        logtracing.AppendSpanKVs(ctx, "entity_id", id, "found", result != nil)
        logtracing.EndSpan(ctx, xerr)
    }()

    // Business logic...
    return result, nil
}
```

### Pattern B: List/Query with Metrics

```go
func (s *Service) List(ctx context.Context, params *ListParams) (result *ListResult, xerr error) {
    ctx, _ = logtracing.StartSpan(ctx, "ServiceName.List")
    defer func() {
        count := 0
        hasMore := false
        if result != nil {
            count = len(result.Items)
            hasMore = result.HasMore
        }
        logtracing.AppendSpanKVs(ctx,
            "limit", params.Limit,
            "offset", params.Offset,
            "result_count", count,
            "has_more", hasMore,
        )
        logtracing.EndSpan(ctx, xerr)
    }()

    // Business logic...
    return result, nil
}
```

### Pattern C: Create/Update with Entity ID

```go
func (s *Service) Create(ctx context.Context, req *CreateRequest) (created *Entity, xerr error) {
    ctx, _ = logtracing.StartSpan(ctx, "ServiceName.Create")
    defer func() {
        createdID := ""
        if created != nil {
            createdID = created.ID
        }
        logtracing.AppendSpanKVs(ctx, "created_id", createdID)
        logtracing.EndSpan(ctx, xerr)
    }()

    // Business logic...
    return created, nil
}
```

### Pattern D: Handler with Request Context

```go
func (h *Handler) HandleCreate(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    ctx, _ = logtracing.StartSpan(ctx, "handler.HandleCreate")
    var createdID string
    defer func() {
        logtracing.AppendSpanKVs(ctx, "created_id", createdID)
        logtracing.EndSpan(ctx, nil)
    }()

    // Parse request...
    logtracing.AppendSpanKVs(ctx, "request_field", req.SomeField)

    // After creation...
    createdID = entity.ID
    // ...
}
```

### Pattern E: Cron/Background Job

```go
func (s *Service) RunScheduledJob(ctx context.Context) (xerr error) {
    ctx, _ = logtracing.StartSpan(ctx, "ServiceName.RunScheduledJob")
    startTime := time.Now()
    var processedCount int
    defer func() {
        logtracing.AppendSpanKVs(ctx,
            "job_name", "scheduled_job_name",
            "execution_time", startTime.Format(time.RFC3339),
            "processed_count", processedCount,
            "duration_ms", time.Since(startTime).Milliseconds(),
        )
        logtracing.EndSpan(ctx, xerr)
    }()

    // Job logic...
    processedCount = len(items)
    return nil
}
```

---

## Quick Reference

### Span Naming

| Format             | Example                | Use Case         |
| ------------------ | ---------------------- | ---------------- |
| `package.Function` | `executor.SelectItems` | Regular function |
| `Type.Method`      | `ItemService.GetByID`  | Service method   |
| `handler.Name`     | `handler.CreateItem`   | HTTP handler     |

### Common Metric Keys

| Category   | Keys                                                               |
| ---------- | ------------------------------------------------------------------ |
| Identity   | `entity_id`, `resource_id`, `request_id`, `created_id`             |
| Count      | `result_count`, `processed_count`, `success_count`, `failed_count` |
| Pagination | `limit`, `offset`, `cursor`, `has_more`, `total_count`             |
| Job/Cron   | `job_name`, `execution_time`, `duration_ms`                        |
| Error      | `error_stage`, `error_code`, `retry_attempt`                       |

### Component Integration Snippets

**GORM**:

```go
gormx.WithSpanName("repo.GetItem")(db).WithContext(ctx).First(&item)
```

**HTTP Client**:

```go
client := &http.Client{Transport: &logtracing.HTTPTransport{Transport: http.DefaultTransport}}
```

**gRPC Server**:

```go
grpc.NewServer(grpc.UnaryInterceptor(logtracing.UnaryServerInterceptor()))
```

**gRPC Client**:

```go
grpc.Dial(addr, grpc.WithUnaryInterceptor(logtracing.UnaryClientInterceptor()))
```

---

## Troubleshooting

| Problem                 | Solution                                                   |
| ----------------------- | ---------------------------------------------------------- |
| No log output           | Ensure `slogx.SetupDefaultLogger` called before logtracing |
| Spans not linked        | Use ctx returned by `StartSpan`                            |
| Error not captured      | Use named return value `xerr`                              |
| Span has no useful info | Add `AppendSpanKVs` with business context                  |
