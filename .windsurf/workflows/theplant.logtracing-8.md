---
description: Implement distributed tracing using logtracing from theplant/appkit for Go services.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Automatically integrate `github.com/theplant/appkit/logtracing` into the project with **standardized span naming, types, and attributes** that enable effective error troubleshooting.

## Execution Steps

### Phase 1: Project Analysis

1. Read `go.mod` to get module name and check dependencies
2. Find main entry point (`cmd/*/main.go` or `main.go`)
3. Find config files (`deploy/`, `config/`, or root)
4. **Analyze project structure to identify layers**:
   - Handler layer (HTTP/gRPC entry points)
   - Service layer (business logic)
   - Repository layer (data access)
   - External service clients (AWS, Firebase, payment gateways, etc.)
   - Queue workers/consumers
   - Cron jobs

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

   - If missing, create in app config directory

5. **Check main.go** for logger init before other components

### Phase 3: Add Tracing by Layer

For each function WITHOUT `logtracing.StartSpan`, add tracing based on its layer:

1. **Handler Layer** → Use Handler Template
2. **Service Layer** → Use Service Template
3. **Repository Layer** → Use Repository Template
4. **External Service** → Use External Service Template
5. **Queue Worker** → Use Queue Worker Template
6. **Cron Job** → Use Cron Job Template

### Phase 4: Verification

1. Run `go build` to verify
2. Run `go mod tidy`
3. Report: files modified, functions traced

## Execution Rules

- Process one file at a time
- Always add imports when adding logtracing
- Preserve existing code logic
- Skip functions with existing tracing
- Skip `*_test.go` and generated files
- **Always add `defer logtracing.RecordPanic(ctx)`**

---

## Span Naming Convention

**Format**: `{scope}/{component}.{operation}`

| Layer            | Format                             | Example                                         |
| ---------------- | ---------------------------------- | ----------------------------------------------- |
| Handler          | `handler/{name}`                   | `handler/CreateOrder`                           |
| Service          | `{package}/{Type}.{Method}`        | `order/OrderService.Create`                     |
| Repository       | `{package}/repository.{Operation}` | `order/repository.FindByID`                     |
| External Service | `{provider}/{service}.{operation}` | `aws/S3.PutObject`, `firebase/auth.CustomToken` |
| Queue            | `job/{domain}.{action}`            | `job/order.ProcessPayment`                      |
| Cron             | `cron/{name}`                      | `cron/daily_report`                             |

---

## Required Span Attributes

**Every span MUST include `span.type` and `span.role`.**

### span.type Values

| Value                | Use Case                 |
| -------------------- | ------------------------ |
| `http`               | HTTP handler             |
| `grpc`               | gRPC handler             |
| `sql`                | Database query           |
| `queue`              | Message queue operation  |
| `cron`               | Scheduled job            |
| `cache`              | Cache operation          |
| `internal`           | Internal service call    |
| `aws.s3`             | AWS S3                   |
| `aws.ses`            | AWS SES                  |
| `aws.kms`            | AWS KMS                  |
| `firebase.auth`      | Firebase Auth            |
| `firebase.messaging` | Firebase Cloud Messaging |
| `gmo`                | GMO Payment Gateway      |

### span.role Values

| Value      | Use Case                       |
| ---------- | ------------------------------ |
| `server`   | Receiving request (handler)    |
| `client`   | Making request (external call) |
| `producer` | Producing message              |
| `consumer` | Consuming message              |
| `job`      | Background job/cron            |
| `internal` | Internal service call          |

---

## Attribute Naming Convention

**Format**: `{domain}.{attribute}` using snake_case

| Domain     | Attributes                                           |
| ---------- | ---------------------------------------------------- |
| `sql`      | `sql.query`, `sql.rows_affected`, `sql.table`        |
| `http`     | `http.method`, `http.path`, `http.status_code`       |
| `queue`    | `queue.name`, `queue.job_id`, `queue.job_latency_ms` |
| `s3`       | `s3.bucket`, `s3.key`, `s3.content_length`           |
| `{entity}` | `order.code`, `user.id`, `product.sku`               |

### Sensitive Data Rules

```go
// ❌ NEVER record sensitive data
logtracing.AppendSpanKVs(ctx, "user.password", password)
logtracing.AppendSpanKVs(ctx, "card.number", cardNumber)

// ✅ Use ID references or masked values
logtracing.AppendSpanKVs(ctx, "user.id", userID)
logtracing.AppendSpanKVs(ctx, "card.last_four", lastFour)
```

---

## Code Templates

### Handler Template (HTTP)

```go
func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    ctx, _ = logtracing.StartSpan(ctx, "handler/CreateOrder")
    defer func() { logtracing.EndSpan(ctx, nil) }()
    defer logtracing.RecordPanic(ctx)
    logtracing.AppendSpanKVs(ctx,
        "span.type", "http",
        "span.role", "server",
    )

    // After parsing request
    logtracing.AppendSpanKVs(ctx,
        "order.user_id", req.UserID,
        "order.items_count", len(req.Items),
    )

    // After creating order
    logtracing.AppendSpanKVs(ctx, "order.code", order.Code)
    // ...
}
```

### Service Template

```go
func (s *OrderService) Create(ctx context.Context, req *CreateRequest) (order *Order, err error) {
    ctx, _ = logtracing.StartSpan(ctx, "order/OrderService.Create")
    defer func() { logtracing.EndSpan(ctx, err) }()
    defer logtracing.RecordPanic(ctx)
    logtracing.AppendSpanKVs(ctx,
        "span.type", "internal",
        "span.role", "internal",
        "order.user_id", req.UserID,
        "order.items_count", len(req.Items),
    )

    // Business logic...

    logtracing.AppendSpanKVs(ctx, "order.code", order.Code)
    return order, nil
}
```

### Repository Template

```go
func (r *OrderRepository) FindByID(ctx context.Context, id string) (order *Order, err error) {
    ctx, _ = logtracing.StartSpan(ctx, "order/repository.FindByID")
    defer func() { logtracing.EndSpan(ctx, err) }()
    defer logtracing.RecordPanic(ctx)
    logtracing.AppendSpanKVs(ctx,
        "span.type", "sql",
        "span.role", "client",
        "order.id", id,
    )

    // Query logic...

    logtracing.AppendSpanKVs(ctx, "sql.found", order != nil)
    return order, nil
}
```

### External Service Template

```go
func (c *S3Client) PutObject(ctx context.Context, bucket, key string, data []byte) (err error) {
    ctx, _ = logtracing.StartSpan(ctx, "aws/S3.PutObject")
    defer func() { logtracing.EndSpan(ctx, err) }()
    defer logtracing.RecordPanic(ctx)
    logtracing.AppendSpanKVs(ctx,
        "span.type", "aws.s3",
        "span.role", "client",
        "s3.bucket", bucket,
        "s3.key", key,
        "s3.content_length", len(data),
    )

    // Call S3...
    return nil
}
```

### Queue Worker Template

```go
func (w *OrderWorker) ProcessPayment(ctx context.Context, job que.Job) (err error) {
    ctx, _ = logtracing.StartSpan(ctx, "job/order.ProcessPayment")
    defer func() { logtracing.EndSpan(ctx, err) }()
    defer logtracing.RecordPanic(ctx)
    logtracing.AppendSpanKVs(ctx,
        "span.type", "queue",
        "span.role", "consumer",
        "queue.job_id", job.ID(),
        "queue.job_args", string(job.Plan().Args),
        "queue.job_latency_ms", time.Since(job.Plan().RunAt).Milliseconds(),
    )

    // Process logic...
    return nil
}
```

### Cron Job Template

```go
func (s *ReportService) GenerateDailyReport(ctx context.Context) (err error) {
    ctx, _ = logtracing.StartSpan(ctx, "cron/daily_report")
    startTime := time.Now()
    var processedCount int
    defer func() {
        logtracing.AppendSpanKVs(ctx,
            "cron.processed_count", processedCount,
            "cron.duration_ms", time.Since(startTime).Milliseconds(),
        )
        logtracing.EndSpan(ctx, err)
    }()
    defer logtracing.RecordPanic(ctx)
    logtracing.AppendSpanKVs(ctx,
        "span.type", "cron",
        "span.role", "job",
        "cron.name", "daily_report",
        "cron.execution_date", time.Now().Format("2006-01-02"),
    )

    // Job logic...
    processedCount = len(items)
    return nil
}
```

### TraceFunc Pattern (Simple Operations)

```go
err = logtracing.TraceFunc(ctx, "firebase/auth.CustomToken", func(ctx context.Context) error {
    logtracing.AppendSpanKVs(ctx,
        "span.type", "firebase.auth",
        "span.role", "client",
        "auth.uid", uid,
    )
    // Call Firebase...
    return nil
})
```

---

## Error Handling

### Standard Pattern

```go
defer func() { logtracing.EndSpan(ctx, err) }()
defer logtracing.RecordPanic(ctx)
```

### Filter Non-Error Responses

```go
defer func() {
    if err != nil {
        var apiErr smithy.APIError
        if errors.As(err, &apiErr) && apiErr.ErrorCode() == "NotModified" {
            logtracing.EndSpan(ctx, nil) // 304 is not an error
            return
        }
    }
    logtracing.EndSpan(ctx, err)
}()
```

---

## Critical Rules

| Rule             | Description                                           |
| ---------------- | ----------------------------------------------------- |
| Named return     | **MUST** use named error return for defer capture     |
| Pass new ctx     | **MUST** pass ctx returned by `StartSpan` to children |
| Defer EndSpan    | **MUST** call `EndSpan` in defer                      |
| RecordPanic      | **MUST** add `defer logtracing.RecordPanic(ctx)`      |
| span.type        | **MUST** set appropriate span type                    |
| span.role        | **MUST** set appropriate span role                    |
| Attribute naming | **MUST** use `{domain}.{attribute}` format            |
| Sensitive data   | **NEVER** log passwords, tokens, card numbers         |

---

## Layer Strategy Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ HTTP/gRPC Handler (Entry Layer)                             │
│ span.type: http/grpc, span.role: server                     │
│ Attributes: request_id, user_id, method, path               │
├─────────────────────────────────────────────────────────────┤
│ Service Layer (Business Logic)                              │
│ span.type: internal, span.role: internal                    │
│ Attributes: Business entity IDs, operation params           │
├─────────────────────────────────────────────────────────────┤
│ Repository Layer (Data Access)                              │
│ span.type: sql, span.role: client                           │
│ Attributes: sql.query, sql.rows_affected                    │
├─────────────────────────────────────────────────────────────┤
│ External Services                                           │
│ span.type: {provider}.{service}, span.role: client          │
│ Attributes: Service-specific params                         │
└─────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting

| Problem                | Solution                                                   |
| ---------------------- | ---------------------------------------------------------- |
| No log output          | Ensure `slogx.SetupDefaultLogger` called before logtracing |
| Spans not linked       | Use ctx returned by `StartSpan`                            |
| Error not captured     | Use named return value                                     |
| Missing span.type/role | Add required attributes                                    |
| Panic not recorded     | Add `defer logtracing.RecordPanic(ctx)`                    |
