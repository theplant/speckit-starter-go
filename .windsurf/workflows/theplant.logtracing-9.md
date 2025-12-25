---
description: Implement simplified distributed tracing using logtracing from theplant/appkit for Go services.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Automatically integrate `github.com/theplant/appkit/logtracing` into the project with **simplified span naming conventions** that reduce boilerplate code while maintaining effective error troubleshooting capabilities.

## Key Improvements from v8

- **Removed mandatory `span.type` and `span.role` attributes** - these are now inferred from span name prefixes
- **70% less boilerplate code** - no repetitive attribute setting in every function
- **Clearer span naming** - prefix-based convention makes layer identification obvious
- **Focus on business context** - only record attributes that aid debugging

## Execution Steps

### Phase 1: Project Analysis

1. Read `go.mod` to get module name and check dependencies
2. Find main entry point (`cmd/*/main.go` or `main.go`)
3. Find config files (`deploy/`, `config/`, or root)
4. **Analyze project structure to identify layers**:
   - Handler layer (HTTP/gRPC entry points)
   - Service layer (business logic)
   - Repository layer (data access)
   - gRPC client calls (external service calls)
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
4. **gRPC Client** → Use gRPC Client Template
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
- **DO NOT add `span.type` or `span.role` attributes** - they are inferred from span name prefix

---

## Span Naming Convention (Type Inference)

**Format**: `{prefix}/{component}.{operation}`

The **prefix automatically implies the span type and role**, eliminating the need for explicit attributes.

| Prefix     | Implied Type | Implied Role | Format                      | Example                           |
| ---------- | ------------ | ------------ | --------------------------- | --------------------------------- |
| `handler/` | `http`       | `server`     | `handler/{FunctionName}`    | `handler/GetMember`               |
| `service/` | `internal`   | `internal`   | `service/{Type}.{Method}`   | `service/MemberService.GetMember` |
| `repo/`    | `sql`        | `client`     | `repo/{table}.{operation}`  | `repo/members.FindByID`           |
| `grpc/`    | `grpc`       | `client`     | `grpc/{service}.{method}`   | `grpc/ledger.GetBalance`          |
| `cache/`   | `cache`      | `client`     | `cache/{store}.{operation}` | `cache/redis.Get`                 |
| `job/`     | `queue`      | `consumer`   | `job/{domain}.{action}`     | `job/order.ProcessPayment`        |
| `cron/`    | `cron`       | `job`        | `cron/{name}`               | `cron/daily_report`               |

**Benefits**:

- Span name alone reveals the layer and type
- No repetitive attribute setting
- Log analysis tools can categorize by prefix
- Cleaner, more readable code

---

## Span Attributes Guidelines

### What to Record

**Only record attributes that help with debugging and troubleshooting:**

| Category                | Attributes                                    | Example                                   |
| ----------------------- | --------------------------------------------- | ----------------------------------------- |
| **Business Entity IDs** | `{entity}.id`                                 | `member.id`, `transaction.id`, `order.id` |
| **Query Operations**    | `query.limit`, `query.filter`, `result.count` | For list/search operations                |
| **Write Operations**    | `operation`, `rows_affected`                  | For create/update/delete                  |
| **External Calls**      | Service-specific params                       | `bucket`, `key`, `user_id`                |
| **Error Context**       | `error.code`, `error.type`                    | When handling specific errors             |

### What NOT to Record

- ❌ `span.type` and `span.role` - inferred from span name prefix
- ❌ HTTP method/path - already in HTTP access logs
- ❌ All request parameters - only record key identifiers
- ❌ Sensitive data - passwords, tokens, card numbers, etc.

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
func (h *Handler) GetMember(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    ctx, _ = logtracing.StartSpan(ctx, "handler/GetMember")
    defer func() { logtracing.EndSpan(ctx, nil) }()
    defer logtracing.RecordPanic(ctx)

    memberID := r.PathValue("memberId")
    logtracing.AppendSpanKVs(ctx, "member.id", memberID)

    // Business logic...
}
```

**Key Points**:

- Span name starts with `handler/` prefix
- No `span.type` or `span.role` needed
- Only record business entity IDs

### Service Template

```go
func (s *MemberService) GetMember(ctx context.Context, memberID string) (member *Member, err error) {
    ctx, _ = logtracing.StartSpan(ctx, "service/MemberService.GetMember")
    defer func() { logtracing.EndSpan(ctx, err) }()
    defer logtracing.RecordPanic(ctx)

    logtracing.AppendSpanKVs(ctx, "member.id", memberID)

    // Business logic...
    return member, nil
}
```

**Key Points**:

- Span name starts with `service/` prefix
- Use named error return for defer capture
- Record key business parameters

### Service Template (List Operations)

```go
func (s *MemberService) ListMembers(ctx context.Context, params *ListParams) (result *ListResult, err error) {
    ctx, _ = logtracing.StartSpan(ctx, "service/MemberService.ListMembers")
    defer func() { logtracing.EndSpan(ctx, err) }()
    defer logtracing.RecordPanic(ctx)

    // Optional: Record query parameters if helpful for debugging
    logtracing.AppendSpanKVs(ctx,
        "query.limit", params.Limit,
        "query.has_filter", params.Query != "",
    )

    // Business logic...

    // Optional: Record result count
    logtracing.AppendSpanKVs(ctx, "result.count", len(result.Items))
    return result, nil
}
```

### Repository Template

```go
func (r *MemberRepository) FindByID(ctx context.Context, id string) (member *Member, err error) {
    ctx, _ = logtracing.StartSpan(ctx, "repo/members.FindByID")
    defer func() { logtracing.EndSpan(ctx, err) }()
    defer logtracing.RecordPanic(ctx)

    logtracing.AppendSpanKVs(ctx, "member.id", id)

    // Query logic...
    return member, nil
}
```

### gRPC Client Template

```go
func (c *LedgerClient) GetBalance(ctx context.Context, req *ledgerpb.GetBalanceRequest) (resp *ledgerpb.GetBalanceResponse, err error) {
    ctx, _ = logtracing.StartSpan(ctx, "grpc/ledger.GetBalance")
    defer func() { logtracing.EndSpan(ctx, err) }()
    defer logtracing.RecordPanic(ctx)

    logtracing.AppendSpanKVs(ctx,
        "user.id", req.UserId,
        "reward_types", strings.Join(req.RewardTypes, ","),
    )

    // gRPC call...
    resp, err = c.client.GetBalance(ctx, req)
    return resp, err
}
```

**Key Points**:

- Span name starts with `grpc/` prefix
- Record key request parameters
- Useful for tracking external service calls

### Queue Worker Template

```go
func (w *OrderWorker) ProcessPayment(ctx context.Context, job que.Job) (err error) {
    ctx, _ = logtracing.StartSpan(ctx, "job/order.ProcessPayment")
    defer func() { logtracing.EndSpan(ctx, err) }()
    defer logtracing.RecordPanic(ctx)

    logtracing.AppendSpanKVs(ctx,
        "job.id", job.ID(),
        "job.latency_ms", time.Since(job.Plan().RunAt).Milliseconds(),
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
            "processed_count", processedCount,
            "duration_ms", time.Since(startTime).Milliseconds(),
        )
        logtracing.EndSpan(ctx, err)
    }()
    defer logtracing.RecordPanic(ctx)

    // Job logic...
    processedCount = len(items)
    return nil
}
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

| Rule               | Description                                                    |
| ------------------ | -------------------------------------------------------------- |
| Named return       | **MUST** use named error return for defer capture              |
| Pass new ctx       | **MUST** pass ctx returned by `StartSpan` to children          |
| Defer EndSpan      | **MUST** call `EndSpan` in defer                               |
| RecordPanic        | **MUST** add `defer logtracing.RecordPanic(ctx)`               |
| Span prefix        | **MUST** use appropriate prefix (`handler/`, `service/`, etc.) |
| No span.type/role  | **DO NOT** add `span.type` or `span.role` attributes           |
| Minimal attributes | Only record attributes that aid debugging                      |
| Sensitive data     | **NEVER** log passwords, tokens, card numbers                  |

---

## Layer Strategy Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ HTTP Handler (Entry Layer)                                  │
│ Span: handler/{FunctionName}                                │
│ Attributes: Business entity IDs only                        │
├─────────────────────────────────────────────────────────────┤
│ Service Layer (Business Logic)                              │
│ Span: service/{ServiceName}.{Method}                        │
│ Attributes: Business entity IDs, key operation params       │
├─────────────────────────────────────────────────────────────┤
│ Repository Layer (Data Access)                              │
│ Span: repo/{table}.{operation}                              │
│ Attributes: Entity IDs, query conditions                    │
├─────────────────────────────────────────────────────────────┤
│ gRPC Clients (External Services)                            │
│ Span: grpc/{service}.{method}                               │
│ Attributes: Request IDs, key parameters                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Migration from v8

If you have existing code using logtracing v8 workflow:

1. **Remove all `span.type` and `span.role` attributes**:

   ```go
   // ❌ Remove these lines
   logtracing.AppendSpanKVs(ctx,
       "span.type", "http",
       "span.role", "server",
   )
   ```

2. **Ensure span names use correct prefixes**:

   ```go
   // ✅ Correct
   logtracing.StartSpan(ctx, "handler/GetMember")
   logtracing.StartSpan(ctx, "service/MemberService.GetMember")
   logtracing.StartSpan(ctx, "grpc/ledger.GetBalance")
   ```

3. **Keep business entity IDs**:
   ```go
   // ✅ Keep these
   logtracing.AppendSpanKVs(ctx, "member.id", memberID)
   logtracing.AppendSpanKVs(ctx, "transaction.id", txID)
   ```

---

## Troubleshooting

| Problem            | Solution                                                   |
| ------------------ | ---------------------------------------------------------- |
| No log output      | Ensure `slogx.SetupDefaultLogger` called before logtracing |
| Spans not linked   | Use ctx returned by `StartSpan`                            |
| Error not captured | Use named return value                                     |
| Wrong span prefix  | Use correct prefix for layer (handler/, service/, etc.)    |
| Panic not recorded | Add `defer logtracing.RecordPanic(ctx)`                    |

---

## Examples: Before and After

### Before (v8 - Verbose)

```go
func (h *MemberHandler) GetMember(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    ctx, _ = logtracing.StartSpan(ctx, "handler/GetMember")
    defer func() { logtracing.EndSpan(ctx, nil) }()
    defer logtracing.RecordPanic(ctx)
    logtracing.AppendSpanKVs(ctx,
        "span.type", "http",      // Redundant
        "span.role", "server",    // Redundant
    )

    memberID := r.PathValue("memberId")
    logtracing.AppendSpanKVs(ctx, "member.id", memberID)
    // ...
}
```

### After (v9 - Simplified)

```go
func (h *MemberHandler) GetMember(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    ctx, _ = logtracing.StartSpan(ctx, "handler/GetMember")
    defer func() { logtracing.EndSpan(ctx, nil) }()
    defer logtracing.RecordPanic(ctx)

    memberID := r.PathValue("memberId")
    logtracing.AppendSpanKVs(ctx, "member.id", memberID)
    // ...
}
```

**Result**: 30% less code, same debugging capability.
