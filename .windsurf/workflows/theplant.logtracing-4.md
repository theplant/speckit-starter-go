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
2. Add tracing to Services and Handlers that are missing it

## Execution Steps

### Phase 1: Project Analysis

1. Read `go.mod` to get module name and check dependencies
2. Find main entry point (`cmd/*/main.go` or `main.go`)
3. Find config files (`deploy/`, `config/`, or root)
4. Identify directories: `services/`, `handlers/`, `internal/app/`, `pkg/`

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

   - If missing, create in `internal/app/`:

   ```go
   var SetupLogger = []any{
       func(conf *Config) *slogx.Config { return conf.Logging },
       slogx.SetupDefaultLogger,
   }
   ```

5. **Check main.go** for logger init before other components

### Phase 3: Service Layer Tracing

1. Find files in `services/`, `internal/services/`, `pkg/*/`
2. For each function with `ctx context.Context` parameter WITHOUT `logtracing.StartSpan`:
   - Add tracing using **Pattern A** or **Pattern B**
   - Ensure error return is named `xerr`
   - Add import if missing

### Phase 4: Handler Layer Tracing

1. Find files in `handlers/`, `internal/handlers/`
2. For each HTTP handler WITHOUT `logtracing.StartSpan`:
   - Add tracing using **Handler Pattern**
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

## Critical Rules

| Rule             | Description                                             |
| ---------------- | ------------------------------------------------------- |
| Named return     | **MUST** use `xerr` for error capture in defer          |
| Pass new ctx     | **MUST** pass ctx returned by `StartSpan` to children   |
| Defer EndSpan    | **MUST** call `EndSpan` in defer                        |
| Setup slog first | **MUST** setup slog before using logtracing             |
| Span naming      | Use `PackageName.FunctionName` or `TypeName.MethodName` |
| Key naming       | **MUST** use snake_case: `user_id`, not `userId`        |

### Common Mistakes

```go
// ❌ Wrong: Ignoring returned ctx
_, _ = logtracing.StartSpan(ctx, "MySpan")

// ✅ Correct
ctx, _ = logtracing.StartSpan(ctx, "MySpan")
```

```go
// ❌ Wrong: Non-named return (err always nil in defer)
func DoWork(ctx context.Context) error {
    var err error
    defer func() { logtracing.EndSpan(ctx, err) }()
    return errors.New("error")
}

// ✅ Correct: Named return
func DoWork(ctx context.Context) (xerr error) {
    defer func() { logtracing.EndSpan(ctx, xerr) }()
    return errors.New("error")
}
```

---

## Code Templates

### Pattern A: Basic Tracing

```go
func (s *Service) MethodName(ctx context.Context, params *Params) (xerr error) {
    ctx, _ = logtracing.StartSpan(ctx, "ServiceName.MethodName")
    defer func() {
        logtracing.EndSpan(ctx, xerr)
    }()

    // Business logic...
    return nil
}
```

### Pattern B: With Metrics

```go
func (s *Service) FetchUsers(ctx context.Context, params *Params) (result *Result, xerr error) {
    ctx, _ = logtracing.StartSpan(ctx, "ServiceName.FetchUsers")
    defer func() {
        count := 0
        if result != nil {
            count = len(result.Users)
        }
        logtracing.AppendSpanKVs(ctx,
            "requested_count", len(params.IDs),
            "fetched_count", count,
        )
        logtracing.EndSpan(ctx, xerr)
    }()

    // Business logic...
    return result, nil
}
```

### Handler Pattern

```go
func (h *Handler) HandleXxx(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    ctx, _ = logtracing.StartSpan(ctx, "handler.HandleXxx")
    defer func() {
        logtracing.EndSpan(ctx, nil)
    }()

    // Use ctx for all subsequent calls
    // ...
}
```

---

## Quick Reference

### Span Naming

| Format             | Example                | Use Case         |
| ------------------ | ---------------------- | ---------------- |
| `package.Function` | `executor.SelectUsers` | Regular function |
| `Type.Method`      | `UserService.GetByID`  | Service method   |
| `handler.Name`     | `handler.CreateOrder`  | HTTP handler     |

### Common Metric Keys

| Category   | Keys                                                                |
| ---------- | ------------------------------------------------------------------- |
| Identity   | `user_id`, `order_id`, `campaign_id`, `transaction_id`              |
| Count      | `requested_count`, `fetched_count`, `success_count`, `failed_count` |
| Pagination | `limit`, `after_cursor`, `has_next`, `total_count`                  |
| Error      | `error_stage`, `invalid_field`, `retry_attempt`                     |

### Component Integration Snippets

**GORM**:

```go
gormx.WithSpanName("repo.GetUser")(db).WithContext(ctx).First(&user)
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

| Problem            | Solution                                                   |
| ------------------ | ---------------------------------------------------------- |
| No log output      | Ensure `slogx.SetupDefaultLogger` called before logtracing |
| Spans not linked   | Use ctx returned by `StartSpan`                            |
| Error not captured | Use named return value `xerr`                              |
