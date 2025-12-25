---
description: Implement distributed tracing using logtracing from theplant/appkit with intelligent attribute selection for Go services.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Integrate `github.com/theplant/appkit/logtracing` with **intelligent attribute selection** - analyzing method logic to determine which parameters should be logged for effective debugging.

## Key Principles

- **Intelligent attribute selection** - analyze method logic to determine what to log
- **Three-step analysis process** - identify essential, helpful, and deduplicated attributes
- **Minimal but sufficient** - log only what's needed to distinguish requests and debug issues

---

## Attribute Selection: Three-Step Analysis

Before adding `logtracing.AppendSpanKVs` to any method, **MUST** perform this analysis:

### Step 1: Identify Essential Information (必须信息)

Analyze the method and identify parameters/variables **essential for distinguishing each request**:

- **Entity IDs**: `member.id`, `order.id`, `transaction.id` - uniquely identifies the request
- **Key input parameters**: Primary function arguments that affect behavior
- **Decision variables**: Status, type, action - determines which code path is taken

### Step 2: Identify Helpful Debug Information (辅助排查信息)

Identify information that **helps troubleshoot issues** but isn't strictly required:

- **Batch operations**: Iteration count, batch size, processed count, success/fail counts
- **Timing information**: Duration for long-running or external operations
- **Conditional results**: Which branch was taken, filter results
- **External call context**: Service name, endpoint, retry count

### Step 3: Deduplicate and Minimize (去重精简)

Review attributes from Steps 1 and 2, remove duplicates and redundant information:

- **Already in parent span**: Don't repeat IDs logged at entry point
- **Derivable information**: Don't log if computable from other logged values
- **Already in HTTP access logs**: Don't duplicate method/path in handler spans
- **Too verbose**: Use counts instead of full arrays

---

## Span Naming Convention

**Format**: `{prefix}/{component}.{operation}`

| Prefix     | Format                      | Example                           |
| ---------- | --------------------------- | --------------------------------- |
| `handler/` | `handler/{FunctionName}`    | `handler/GetMember`               |
| `service/` | `service/{Type}.{Method}`   | `service/MemberService.GetMember` |
| `repo/`    | `repo/{table}.{operation}`  | `repo/members.FindByID`           |
| `grpc/`    | `grpc/{service}.{method}`   | `grpc/ledger.GetBalance`          |
| `cache/`   | `cache/{store}.{operation}` | `cache/redis.Get`                 |
| `job/`     | `job/{domain}.{action}`     | `job/order.ProcessPayment`        |
| `cron/`    | `cron/{name}`               | `cron/daily_report`               |

---

## Code Templates

### Handler Template

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

### Service Template - Batch Operations

```go
func (s *SyncService) SyncMembers(ctx context.Context, memberIDs []string) (err error) {
    ctx, _ = logtracing.StartSpan(ctx, "service/SyncService.SyncMembers")
    startTime := time.Now()
    var successCount, failCount int

    defer func() {
        logtracing.AppendSpanKVs(ctx,
            "batch.total", len(memberIDs),
            "batch.success", successCount,
            "batch.failed", failCount,
            "duration_ms", time.Since(startTime).Milliseconds(),
        )
        logtracing.EndSpan(ctx, err)
    }()
    defer logtracing.RecordPanic(ctx)

    // Process logic...
    return nil
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
    return nil
}
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

| Category                                              | Reason                 |
| ----------------------------------------------------- | ---------------------- |
| Sensitive data (passwords, tokens, card numbers)      | Security               |
| Large payloads (full request/response bodies)         | Performance            |
| Redundant IDs (same ID at every layer)                | Noise                  |
| HTTP metadata (method, path, headers)                 | Already in access logs |
| Derivable values (total when you have success+failed) | Redundancy             |
| Array contents (full list of IDs)                     | Use count instead      |

---

## Execution Steps

### Phase 1: Project Analysis

1. Read `go.mod` to get module name and check dependencies
2. Find main entry point (`cmd/*/main.go` or `main.go`)
3. Find config files (`deploy/`, `config/`, or root)
4. Identify layers: Handler, Service, Repository, gRPC client, Queue workers, Cron jobs

### Phase 2: Dependencies & Configuration

1. Check `go.mod` for `github.com/theplant/appkit/logtracing` and `github.com/qor5/x/v3/slogx`, add if missing
2. Check Config struct for `Logging *slogx.Config`, add if missing
3. Check YAML config for `logging:` section, add if missing
4. Check SetupLogger provider exists
5. Check main.go for logger init before other components

### Phase 3: Add Tracing

For each function WITHOUT `logtracing.StartSpan`:

1. Identify the layer (Handler/Service/Repository/etc.)
2. **Apply Three-Step Analysis** to determine attributes
3. Add tracing code using appropriate template

### Phase 4: Verification

1. Run `go build` to verify
2. Run `go mod tidy`
3. Report: files modified, functions traced

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
