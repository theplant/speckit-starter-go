---
description: Implement distributed tracing using logtracing from theplant/appkit with intelligent attribute selection for Go services.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Automatically integrate `github.com/theplant/appkit/logtracing` into the project with **intelligent attribute selection** - analyzing method logic to determine which parameters and variables should be logged for effective debugging.

## Key Improvements from v9

- **Intelligent attribute selection** - analyze method logic to determine what to log
- **Three-step analysis process** - identify essential, helpful, and deduplicated attributes
- **Context-aware logging** - different methods need different logging strategies
- **Minimal but sufficient** - log only what's needed to distinguish requests and debug issues

---

## Attribute Selection Principles (Core Innovation)

### The Three-Step Analysis Process

Before adding `logtracing.AppendSpanKVs` to any method, perform this analysis:

#### Step 1: Identify Essential Information (必须信息)

Analyze the method logic and identify parameters/variables that are **essential for distinguishing each request**:

| Question to Ask                        | What to Log                                            |
| -------------------------------------- | ------------------------------------------------------ |
| What uniquely identifies this request? | Entity IDs (`member.id`, `order.id`, `transaction.id`) |
| What are the key input parameters?     | Primary function arguments that affect behavior        |
| What determines the code path taken?   | Decision variables (status, type, mode)                |

**Example Analysis**:

```go
// Method: ProcessOrder(ctx, orderID, paymentMethod string, amount float64)
// Step 1 Analysis:
// - orderID: Essential - uniquely identifies the request ✅
// - paymentMethod: Essential - affects processing logic ✅
// - amount: Not essential - can be retrieved from order record ❌
```

#### Step 2: Identify Helpful Debug Information (辅助排查信息)

Identify information that **helps troubleshoot issues** but isn't strictly required:

| Category                  | What to Log                                  | When to Log                             |
| ------------------------- | -------------------------------------------- | --------------------------------------- |
| **Loop/Batch Operations** | Iteration count, batch size, processed count | When processing multiple items          |
| **Timing Information**    | Duration, latency                            | For long-running or external operations |
| **Conditional Results**   | Which branch was taken, filter results       | When debugging logic flow               |
| **External Call Context** | Service name, endpoint, retry count          | For external service calls              |

**Example Analysis**:

```go
// Method: SyncMembers(ctx, memberIDs []string)
// Step 2 Analysis:
// - len(memberIDs): Helpful - shows batch size for performance analysis ✅
// - sync duration: Helpful - identifies slow syncs ✅
// - individual member sync times: Too detailed - skip ❌
```

#### Step 3: Deduplicate and Minimize (去重精简)

Review attributes from Steps 1 and 2, remove duplicates and redundant information:

| Redundancy Type                   | Action                                                   |
| --------------------------------- | -------------------------------------------------------- |
| Same ID logged at multiple layers | Keep only at entry point or where most relevant          |
| Derivable information             | Don't log if it can be computed from other logged values |
| Already in parent span            | Don't repeat in child spans                              |
| Available in HTTP access logs     | Don't duplicate in handler spans                         |

**Example Analysis**:

```go
// Handler logs member.id
// Service also receives memberID
// Repository also receives memberID
//
// Decision: Log member.id only in Handler (entry point)
// Service and Repo can trace back via span hierarchy
```

---

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

### Phase 3: Add Tracing with Intelligent Attribute Selection

For each function WITHOUT `logtracing.StartSpan`:

1. **Identify the layer** (Handler/Service/Repository/etc.)
2. **Apply Three-Step Analysis** to determine attributes
3. **Add tracing code** using appropriate template

### Phase 4: Verification

1. Run `go build` to verify
2. Run `go mod tidy`
3. Report: files modified, functions traced, attributes added

---

## Span Naming Convention

**Format**: `{prefix}/{component}.{operation}`

| Prefix     | Implied Type | Implied Role | Format                      | Example                           |
| ---------- | ------------ | ------------ | --------------------------- | --------------------------------- |
| `handler/` | `http`       | `server`     | `handler/{FunctionName}`    | `handler/GetMember`               |
| `service/` | `internal`   | `internal`   | `service/{Type}.{Method}`   | `service/MemberService.GetMember` |
| `repo/`    | `sql`        | `client`     | `repo/{table}.{operation}`  | `repo/members.FindByID`           |
| `grpc/`    | `grpc`       | `client`     | `grpc/{service}.{method}`   | `grpc/ledger.GetBalance`          |
| `cache/`   | `cache`      | `client`     | `cache/{store}.{operation}` | `cache/redis.Get`                 |
| `job/`     | `queue`      | `consumer`   | `job/{domain}.{action}`     | `job/order.ProcessPayment`        |
| `cron/`    | `cron`       | `job`        | `cron/{name}`               | `cron/daily_report`               |

---

## Code Templates with Attribute Analysis Examples

### Handler Template

```go
func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    ctx, _ = logtracing.StartSpan(ctx, "handler/CreateOrder")
    defer func() { logtracing.EndSpan(ctx, nil) }()
    defer logtracing.RecordPanic(ctx)

    // Three-Step Analysis:
    // Step 1 (Essential): memberID - identifies who is creating the order
    // Step 2 (Helpful): none needed at handler level
    // Step 3 (Dedupe): HTTP method/path already in access logs, skip

    memberID := r.PathValue("memberId")
    logtracing.AppendSpanKVs(ctx, "member.id", memberID)

    // Business logic...
}
```

### Service Template - Simple Operation

```go
func (s *MemberService) GetMember(ctx context.Context, memberID string) (member *Member, err error) {
    ctx, _ = logtracing.StartSpan(ctx, "service/MemberService.GetMember")
    defer func() { logtracing.EndSpan(ctx, err) }()
    defer logtracing.RecordPanic(ctx)

    // Three-Step Analysis:
    // Step 1 (Essential): memberID - the lookup key
    // Step 2 (Helpful): none - simple lookup
    // Step 3 (Dedupe): memberID likely logged in handler, but service may be called directly
    //                  Keep it for standalone service calls

    logtracing.AppendSpanKVs(ctx, "member.id", memberID)

    // Business logic...
    return member, nil
}
```

### Service Template - Complex Business Logic

```go
func (s *OrderService) ProcessOrder(ctx context.Context, orderID string, action string) (err error) {
    ctx, _ = logtracing.StartSpan(ctx, "service/OrderService.ProcessOrder")
    defer func() { logtracing.EndSpan(ctx, err) }()
    defer logtracing.RecordPanic(ctx)

    // Three-Step Analysis:
    // Step 1 (Essential):
    //   - orderID: uniquely identifies the order
    //   - action: determines which code path is taken (approve/reject/cancel)
    // Step 2 (Helpful): none initially, may add based on logic
    // Step 3 (Dedupe): both are essential, no duplicates

    logtracing.AppendSpanKVs(ctx,
        "order.id", orderID,
        "action", action,
    )

    // Business logic with different paths based on action...
    return nil
}
```

### Service Template - Batch/Loop Operations

```go
func (s *SyncService) SyncMembers(ctx context.Context, memberIDs []string) (err error) {
    ctx, _ = logtracing.StartSpan(ctx, "service/SyncService.SyncMembers")
    startTime := time.Now()
    var successCount, failCount int

    defer func() {
        // Three-Step Analysis for defer:
        // Step 2 (Helpful): batch metrics help identify performance issues
        logtracing.AppendSpanKVs(ctx,
            "batch.total", len(memberIDs),
            "batch.success", successCount,
            "batch.failed", failCount,
            "duration_ms", time.Since(startTime).Milliseconds(),
        )
        logtracing.EndSpan(ctx, err)
    }()
    defer logtracing.RecordPanic(ctx)

    // Three-Step Analysis:
    // Step 1 (Essential): batch size (len) - identifies the scope of work
    // Step 2 (Helpful): success/fail counts, duration - for troubleshooting
    // Step 3 (Dedupe): don't log individual memberIDs (too verbose)

    logtracing.AppendSpanKVs(ctx, "batch.total", len(memberIDs))

    for _, memberID := range memberIDs {
        if err := s.syncOneMember(ctx, memberID); err != nil {
            failCount++
            continue
        }
        successCount++
    }
    return nil
}
```

### Service Template - Conditional Logic

```go
func (s *PaymentService) ProcessPayment(ctx context.Context, orderID string, method PaymentMethod) (err error) {
    ctx, _ = logtracing.StartSpan(ctx, "service/PaymentService.ProcessPayment")
    defer func() { logtracing.EndSpan(ctx, err) }()
    defer logtracing.RecordPanic(ctx)

    // Three-Step Analysis:
    // Step 1 (Essential):
    //   - orderID: identifies the order
    //   - method: determines payment gateway used (critical for debugging)
    // Step 2 (Helpful): gateway response code if available
    // Step 3 (Dedupe): amount not needed - can be retrieved from order

    logtracing.AppendSpanKVs(ctx,
        "order.id", orderID,
        "payment.method", string(method),
    )

    switch method {
    case PaymentMethodCard:
        // Card payment logic...
    case PaymentMethodWallet:
        // Wallet payment logic...
    }
    return nil
}
```

### Repository Template

```go
func (r *MemberRepository) FindByID(ctx context.Context, id string) (member *Member, err error) {
    ctx, _ = logtracing.StartSpan(ctx, "repo/members.FindByID")
    defer func() { logtracing.EndSpan(ctx, err) }()
    defer logtracing.RecordPanic(ctx)

    // Three-Step Analysis:
    // Step 1 (Essential): id - the lookup key
    // Step 2 (Helpful): none for simple lookup
    // Step 3 (Dedupe): id likely logged in service, but repo may be reused
    //                  Keep for traceability

    logtracing.AppendSpanKVs(ctx, "member.id", id)

    // Query logic...
    return member, nil
}
```

### Repository Template - List with Filters

```go
func (r *OrderRepository) FindByFilters(ctx context.Context, filters *OrderFilters) (orders []*Order, err error) {
    ctx, _ = logtracing.StartSpan(ctx, "repo/orders.FindByFilters")
    defer func() {
        // Step 2: Result count helps identify empty results vs errors
        logtracing.AppendSpanKVs(ctx, "result.count", len(orders))
        logtracing.EndSpan(ctx, err)
    }()
    defer logtracing.RecordPanic(ctx)

    // Three-Step Analysis:
    // Step 1 (Essential):
    //   - status filter: affects which records are returned
    //   - member_id filter: scopes the query
    // Step 2 (Helpful): result count, has_date_filter (boolean, not actual dates)
    // Step 3 (Dedupe): don't log all filter values, just key ones

    logtracing.AppendSpanKVs(ctx,
        "filter.status", filters.Status,
        "filter.member_id", filters.MemberID,
        "filter.has_date_range", filters.StartDate != nil,
    )

    // Query logic...
    return orders, nil
}
```

### gRPC Client Template

```go
func (c *LedgerClient) GetBalance(ctx context.Context, req *ledgerpb.GetBalanceRequest) (resp *ledgerpb.GetBalanceResponse, err error) {
    ctx, _ = logtracing.StartSpan(ctx, "grpc/ledger.GetBalance")
    defer func() { logtracing.EndSpan(ctx, err) }()
    defer logtracing.RecordPanic(ctx)

    // Three-Step Analysis:
    // Step 1 (Essential): user_id - identifies whose balance
    // Step 2 (Helpful): reward_types - helps debug "wrong balance" issues
    // Step 3 (Dedupe): user_id may be in parent span, but gRPC calls
    //                  are often the source of issues, keep for clarity

    logtracing.AppendSpanKVs(ctx,
        "user.id", req.UserId,
        "reward_types", strings.Join(req.RewardTypes, ","),
    )

    resp, err = c.client.GetBalance(ctx, req)
    return resp, err
}
```

### Queue Worker Template

```go
func (w *OrderWorker) ProcessPayment(ctx context.Context, job que.Job) (err error) {
    ctx, _ = logtracing.StartSpan(ctx, "job/order.ProcessPayment")
    defer func() { logtracing.EndSpan(ctx, err) }()
    defer logtracing.RecordPanic(ctx)

    // Three-Step Analysis:
    // Step 1 (Essential): job.id - identifies the job
    // Step 2 (Helpful): latency - helps identify queue delays
    // Step 3 (Dedupe): job payload details logged when processing

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
        // Step 2: Cron jobs benefit from execution metrics
        logtracing.AppendSpanKVs(ctx,
            "processed_count", processedCount,
            "duration_ms", time.Since(startTime).Milliseconds(),
        )
        logtracing.EndSpan(ctx, err)
    }()
    defer logtracing.RecordPanic(ctx)

    // Three-Step Analysis:
    // Step 1 (Essential): none - cron jobs are identified by span name
    // Step 2 (Helpful): processed_count, duration - for monitoring
    // Step 3 (Dedupe): no duplicates

    // Job logic...
    processedCount = len(items)
    return nil
}
```

---

## Attribute Selection Decision Tree

```
┌─────────────────────────────────────────────────────────────┐
│                    Analyze Method                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 1: What uniquely identifies this request?              │
│ - Entity IDs (member.id, order.id, etc.)                    │
│ - Key input parameters that affect behavior                 │
│ - Decision variables (status, type, action)                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: What helps troubleshoot issues?                     │
│ - Batch operations: count, success/fail                     │
│ - Long operations: duration                                 │
│ - External calls: service context                           │
│ - Conditional logic: which branch taken                     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Remove duplicates and redundancy                    │
│ - Already logged in parent span?                            │
│ - Can be derived from other logged values?                  │
│ - Already in HTTP access logs?                              │
│ - Too verbose (e.g., full arrays)?                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Final Attribute List                           │
│         (Minimal but sufficient for debugging)              │
└─────────────────────────────────────────────────────────────┘
```

---

## Common Patterns and Their Attributes

| Method Pattern        | Essential Attributes          | Helpful Attributes                             |
| --------------------- | ----------------------------- | ---------------------------------------------- |
| **Get by ID**         | `{entity}.id`                 | -                                              |
| **List/Search**       | Key filters                   | `result.count`                                 |
| **Create**            | -                             | Created entity ID (in defer)                   |
| **Update**            | `{entity}.id`, changed fields | -                                              |
| **Delete**            | `{entity}.id`                 | -                                              |
| **Batch Process**     | `batch.total`                 | `batch.success`, `batch.failed`, `duration_ms` |
| **External Call**     | Key request params            | Response status                                |
| **Conditional Logic** | Decision variable             | -                                              |
| **Cron Job**          | -                             | `processed_count`, `duration_ms`               |

---

## What NOT to Log

| Category             | Examples                           | Reason                 |
| -------------------- | ---------------------------------- | ---------------------- |
| **Sensitive Data**   | passwords, tokens, card numbers    | Security               |
| **Large Payloads**   | Full request/response bodies       | Performance            |
| **Redundant IDs**    | Same ID at every layer             | Noise                  |
| **HTTP Metadata**    | Method, path, headers              | Already in access logs |
| **Derivable Values** | Total when you have success+failed | Redundancy             |
| **Array Contents**   | Full list of IDs                   | Use count instead      |

---

## Critical Rules

| Rule                | Description                                                    |
| ------------------- | -------------------------------------------------------------- |
| Named return        | **MUST** use named error return for defer capture              |
| Pass new ctx        | **MUST** pass ctx returned by `StartSpan` to children          |
| Defer EndSpan       | **MUST** call `EndSpan` in defer                               |
| RecordPanic         | **MUST** add `defer logtracing.RecordPanic(ctx)`               |
| Span prefix         | **MUST** use appropriate prefix (`handler/`, `service/`, etc.) |
| Three-Step Analysis | **MUST** analyze before adding attributes                      |
| Minimal attributes  | Only record attributes that aid debugging                      |
| Sensitive data      | **NEVER** log passwords, tokens, card numbers                  |

---

## Execution Rules

- Process one file at a time
- Always add imports when adding logtracing
- Preserve existing code logic
- Skip functions with existing tracing
- Skip `*_test.go` and generated files
- **Always add `defer logtracing.RecordPanic(ctx)`**
- **Apply Three-Step Analysis before adding attributes**

---

## Troubleshooting

| Problem             | Solution                                                   |
| ------------------- | ---------------------------------------------------------- |
| No log output       | Ensure `slogx.SetupDefaultLogger` called before logtracing |
| Spans not linked    | Use ctx returned by `StartSpan`                            |
| Error not captured  | Use named return value                                     |
| Wrong span prefix   | Use correct prefix for layer                               |
| Panic not recorded  | Add `defer logtracing.RecordPanic(ctx)`                    |
| Too many attributes | Re-apply Three-Step Analysis, focus on Step 3              |
| Missing key info    | Re-apply Step 1, ensure essential IDs are logged           |
