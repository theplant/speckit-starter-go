---
description: Implement distributed tracing using logtracing from theplant/appkit for Go services.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Automatically check and integrate `github.com/theplant/appkit/logtracing` into the project. This workflow will:

1. Check dependencies and configuration, add logtracing setup if missing
2. Check Services, Handlers and other components, add tracing if missing

## Execution Steps

### Phase 1: Project Analysis

1. **Analyze project structure**:

   - Find `go.mod` to identify module name and existing dependencies
   - Find main entry point (usually `cmd/*/main.go` or `main.go`)
   - Find config files (YAML/JSON in `deploy/`, `config/`, or project root)
   - Identify project patterns (lifecycle DI vs simple init)

2. **Identify key directories**:
   - `services/` or `internal/services/` - Service layer
   - `handlers/` or `internal/handlers/` - HTTP handlers
   - `internal/app/` - App configuration and providers
   - `pkg/` - Shared packages

### Phase 2: Dependencies & Configuration Check

1. **Check go.mod for required dependencies**:

   ```
   github.com/theplant/appkit/logtracing
   github.com/qor5/x/v3/slogx
   ```

   - If missing, run `go get` to add them

2. **Check for slogx config in Config struct**:

   - Look for `*slogx.Config` field in config struct
   - If missing, add `Logging *slogx.Config \`confx:"logging"\`` field

3. **Check for YAML config**:

   - Look for `logging:` section in config files
   - If missing, add:
     ```yaml
     logging:
       level: "info"
     ```

4. **Check for SetupLogger provider**:

   - Look for `slogx.SetupDefaultLogger` usage
   - If missing, create SetupLogger provider in `internal/app/`

5. **Check main.go for logger initialization**:
   - Ensure SetupLogger is called before other components
   - If using lifecycle, add to `lifecycle.Serve()` chain
   - If simple init, add `slog.SetDefault()` call

### Phase 3: Service Layer Tracing

1. **Find all service files**:

   - Search for files in `services/`, `internal/services/`, `pkg/*/`
   - Identify exported functions with `ctx context.Context` parameter

2. **For each service function WITHOUT tracing**:
   - Check if function already has `logtracing.StartSpan`
   - If missing, add tracing pattern:
     ```go
     func (s *Service) MethodName(ctx context.Context, ...) (result Type, xerr error) {
         ctx, _ = logtracing.StartSpan(ctx, "ServiceName.MethodName")
         defer func() {
             logtracing.EndSpan(ctx, xerr)
         }()
         // existing code...
     }
     ```
   - Ensure return value is named `xerr` for error capture
   - Add `"github.com/theplant/appkit/logtracing"` import if missing

### Phase 4: Handler Layer Tracing

1. **Find all handler files**:

   - Search for files in `handlers/`, `internal/handlers/`
   - Identify HTTP handler functions

2. **For each handler function WITHOUT tracing**:
   - Check if function already has `logtracing.StartSpan`
   - If missing, add tracing pattern:
     ```go
     func (h *Handler) HandleXxx(w http.ResponseWriter, r *http.Request) {
         ctx := r.Context()
         ctx, _ = logtracing.StartSpan(ctx, "handler.HandleXxx")
         defer func() {
             logtracing.EndSpan(ctx, nil)
         }()
         // existing code - update ctx usage
     }
     ```
   - Add `"github.com/theplant/appkit/logtracing"` import if missing

### Phase 5: Component Integrations Check

1. **GORM Integration**:

   - Find GORM database operations
   - Check if using `gormx.WithSpanName`
   - If not, suggest adding tracing wrapper

2. **HTTP Client Integration**:

   - Find `http.Client` usage
   - Check if using `logtracing.HTTPTransport`
   - If not, suggest wrapping with tracing transport

3. **gRPC Integration** (if applicable):
   - Find gRPC server/client setup
   - Check if using logtracing interceptors
   - If not, suggest adding interceptors

### Phase 6: Verification

1. **Run `go build`** to verify no compilation errors
2. **Run `go mod tidy`** to clean up dependencies
3. **Report summary**:
   - List all files modified
   - List all functions with tracing added
   - List any manual actions needed

## Execution Rules

- **Process one file at a time** to avoid conflicts
- **Always add imports** when adding logtracing calls
- **Preserve existing code logic** - only add tracing wrapper
- **Use consistent span naming**: `PackageName.FunctionName` or `TypeName.MethodName`
- **Handle errors properly**: use named return value `xerr`
- **Skip functions that already have tracing**
- **Skip test files** (`*_test.go`)
- **Skip generated files** (files with `// Code generated` comment)

---

## Reference: Prerequisites

- Install dependencies:

```bash
go get github.com/theplant/appkit/logtracing
go get github.com/qor5/x/v3/slogx
```

## Core Concepts

- **Span**: Represents an operation's execution time, including name, start time, duration, and error info
- **Trace**: A chain of Spans linked by `trace.id`
- **Context**: Trace info is passed through Go's `context.Context`

### Auto-recorded Fields

When using `StartSpan` and `EndSpan`, these fields are automatically added:

| Field            | Description        |
| ---------------- | ------------------ |
| `ts`             | Timestamp          |
| `msg`            | Log message        |
| `trace.id`       | Trace ID           |
| `span.id`        | Span ID            |
| `span.context`   | Span name/context  |
| `span.dur_ms`    | Span duration (ms) |
| `span.parent_id` | Parent Span ID     |

If error is recorded:

| Field           | Description    |
| --------------- | -------------- |
| `span.err`      | Error message  |
| `span.err_type` | Error type     |
| `span.with_err` | Has error flag |

## Initialization

### Option 1: With lifecycle dependency injection (Recommended)

#### Step 1: Define config struct

```go
package app

import "github.com/qor5/x/v3/slogx"

type Config struct {
    Logging *slogx.Config `confx:"logging"`
    // Other config fields...
}
```

#### Step 2: Add YAML config

```yaml
logging:
  level: "info" # Options: debug, info, warn, error
```

#### Step 3: Create SetupLogger provider

```go
package app

import "github.com/qor5/x/v3/slogx"

var SetupLogger = []any{
    func(conf *Config) *slogx.Config {
        return conf.Logging
    },
    slogx.SetupDefaultLogger,
}
```

#### Step 4: Inject at service startup

```go
package main

import (
    "context"
    "github.com/theplant/inject/lifecycle"
    "your-project/internal/app"
)

func main() {
    err := lifecycle.Serve(context.Background(),
        lifecycle.SetupSignal,
        // 1. Load config
        func(ctx context.Context) (*app.Config, error) {
            return loadConfig(ctx)
        },
        // 2. Setup logger (MUST be before components using logtracing)
        app.SetupLogger,
        // 3. Other components...
        app.SetupDB,
        app.SetupHTTPServer,
    )
    if err != nil {
        panic(err)
    }
}
```

### Option 2: Simple initialization

```go
package main

import (
    "log/slog"
    "os"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))
    slog.SetDefault(logger)
    // Now logtracing is ready to use
}
```

## Core API

| API                                     | Description                            |
| --------------------------------------- | -------------------------------------- |
| `logtracing.StartSpan(ctx, name)`       | Start a Span, returns new ctx and span |
| `logtracing.EndSpan(ctx, err)`          | End Span, pass error (can be nil)      |
| `logtracing.AppendSpanKVs(ctx, kvs...)` | Append key-value pairs to current Span |
| `logtracing.RecordPanic(ctx)`           | Record panic to Span                   |

## Usage Patterns

### Pattern A: Basic Function Tracing

```go
func DoSomething(ctx context.Context) (xerr error) {
    ctx, _ = logtracing.StartSpan(ctx, "package.DoSomething")
    defer func() {
        logtracing.EndSpan(ctx, xerr)
    }()

    // Business logic...
    return nil
}
```

**Key points**:

- Use **named return value** `xerr` so defer can access the final error
- `StartSpan` returns new ctx, use it for subsequent operations
- `EndSpan` auto-records duration and error info

### Pattern B: With Metrics

```go
func FetchUsers(ctx context.Context, params *Params) (result *Result, xerr error) {
    ctx, _ = logtracing.StartSpan(ctx, "userService.FetchUsers")
    defer func() {
        usersCount := 0
        if result != nil {
            usersCount = len(result.Users)
        }
        logtracing.AppendSpanKVs(ctx,
            "requested_count", len(params.IDs),
            "fetched_count", usersCount,
        )
        logtracing.EndSpan(ctx, xerr)
    }()

    // Business logic...
    return result, nil
}
```

### Pattern C: Using Span Object

```go
func ProcessData(ctx context.Context) (xerr error) {
    ctx, span := logtracing.StartSpan(ctx, "processor.ProcessData")
    defer func() {
        logtracing.EndSpan(ctx, xerr)
    }()

    result := db.Delete(&Model{})
    span.AppendKVs("rows_affected", result.RowsAffected)

    return nil
}
```

### Pattern D: With Panic Recovery

```go
func RiskyOperation(ctx context.Context) (xerr error) {
    ctx, _ = logtracing.StartSpan(ctx, "service.RiskyOperation")
    defer func() {
        logtracing.EndSpan(ctx, xerr)
    }()
    defer logtracing.RecordPanic(ctx)

    // Risky business logic...
    return nil
}
```

## Component Integrations

### GORM Integration

```go
import "github.com/qor5/x/v3/gormx"

func GetUserByID(ctx context.Context, db *gorm.DB, id string) (*User, error) {
    var user User
    err := gormx.WithSpanName("userRepo.GetUserByID")(db).
        WithContext(ctx).
        Where("id = ?", id).
        First(&user).Error
    return &user, err
}
```

### gRPC Integration

```go
import "github.com/theplant/appkit/logtracing"

// Server
server := grpc.NewServer(
    grpc.UnaryInterceptor(logtracing.UnaryServerInterceptor()),
    grpc.StreamInterceptor(logtracing.StreamServerInterceptor()),
)

// Client
conn, err := grpc.Dial(address,
    grpc.WithUnaryInterceptor(logtracing.UnaryClientInterceptor()),
    grpc.WithStreamInterceptor(logtracing.StreamClientInterceptor()),
)
```

### HTTP Client Integration

```go
import "github.com/theplant/appkit/logtracing"

client := &http.Client{
    Transport: &logtracing.HTTPTransport{
        Transport: http.DefaultTransport,
    },
}

// Or use TraceHTTPRequest
resp, err := logtracing.TraceHTTPRequest(
    client.Do,
    "external-api",
    req,
)
```

### HTTP Server Integration

```go
func myHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    ctx, _ = logtracing.StartSpan(ctx, "handler.myHandler")
    defer func() {
        logtracing.EndSpan(ctx, nil)
    }()

    logtracing.AppendSpanKVs(ctx, logtracing.HTTPServerKVs(r)...)
    // Handle request...
}
```

### go-que Queue Integration

```go
import (
    "github.com/qor5/x/v3/goquex"
    "github.com/qor5/go-que"
    "github.com/qor5/go-que/pg"
)

goq, _ := pg.NewWithOptions(pg.Options{DB: sqlDB})
goq = goquex.WithTracing(goq)

worker, err := quex.StartWorker(ctx, que.WorkerOptions{
    Queue:   "my-queue",
    Perform: goquex.PerformWithTracing(notifier)(myPerformFunc),
})

lc.Add(goquex.NewWorkerService(processor.Start).WithName("my-worker"))
```

## Advanced Usage

### Define AppendSpanKVs for Domain Objects

```go
type Task struct {
    ID         string
    CampaignID string
    Status     string
}

func (t *Task) AppendSpanKVs(ctx context.Context, kvs ...any) {
    if t == nil {
        return
    }
    allKVs := append([]any{
        "task.id", t.ID,
        "task.campaign_id", t.CampaignID,
        "task.status", t.Status,
    }, kvs...)
    logtracing.AppendSpanKVs(ctx, allKVs...)
}
```

### Add Common Fields to All Child Spans

```go
func HandleRequest(ctx context.Context, userID string) error {
    ctx = logtracing.ContextWithKVs(ctx, "user_id", userID)
    // All child Spans will include user_id
    return processOrder(ctx)
}
```

### Configure Sampling Rate

```go
import "github.com/theplant/appkit/logtracing"

func init() {
    logtracing.ApplyConfig(logtracing.Config{
        DefaultSampler: logtracing.ProbabilitySampler(0.1), // 10% sampling
    })
}
```

Available samplers:

- `logtracing.AlwaysSample()` - 100% sampling (default)
- `logtracing.NeverSample()` - No sampling
- `logtracing.ProbabilitySampler(fraction)` - Probability sampling

### Export to External Systems (e.g., Honeycomb)

```go
import (
    "github.com/theplant/appkit/logtracing"
    "github.com/theplant/appkit/logtracing/exporters/honeycomb"
    "github.com/honeycombio/libhoney-go"
)

func setupExporter() error {
    exporter, err := honeycomb.NewExporter(libhoney.Config{
        WriteKey: "your-api-key",
        Dataset:  "your-dataset",
    })
    if err != nil {
        return err
    }
    logtracing.RegisterExporter(exporter)
    return nil
}
```

## AI Agent Requirements

### Span Naming Convention

| Format                            | Example                           | Use Case         |
| --------------------------------- | --------------------------------- | ---------------- |
| `package.FunctionName`            | `executor.SelectUsers`            | Regular function |
| `package/subpackage.FunctionName` | `executor/processor.completeTask` | Subpackage func  |
| `type.MethodName`                 | `UserService.GetByID`             | Service method   |
| `Source.MethodName`               | `EventSource.Extract`             | ETL operation    |

### Key Naming Convention

- **MUST use snake_case** for all key names:

```go
// ✅ Correct
logtracing.AppendSpanKVs(ctx, "user_id", userID, "task_status", status)

// ❌ Wrong
logtracing.AppendSpanKVs(ctx, "userId", userID, "taskStatus", status)
```

### Common Metric Keys

| Category     | Key Examples                                                       |
| ------------ | ------------------------------------------------------------------ |
| **Count**    | `selected_count`, `succeed_count`, `failed_count`, `fetched_count` |
| **Perf**     | `task_duration_ms`, `success_rate_percent`                         |
| **Resource** | `bitmap_size_bytes`, `batch_size`                                  |
| **Paging**   | `has_more`, `next_cursor`, `cursor`                                |
| **Identity** | `task_id`, `campaign_id`, `user_id`                                |

### Critical Rules

1. **MUST use named return value** (`xerr`) for error capture in defer
2. **MUST pass the new ctx** returned by `StartSpan` to child functions
3. **MUST call `EndSpan`** in defer to ensure it's always called
4. **MUST setup slog** before using logtracing
5. **SHOULD use `gormx.WithSpanName`** for GORM database operations
6. **SHOULD use interceptors** for gRPC services
7. **SHOULD use `HTTPTransport`** for HTTP clients

### Common Mistakes to Avoid

```go
// ❌ Wrong: Ignoring returned ctx
_, _ = logtracing.StartSpan(ctx, "MySpan")

// ✅ Correct: Use returned ctx
ctx, _ = logtracing.StartSpan(ctx, "MySpan")
```

```go
// ❌ Wrong: Non-named return value
func DoWork(ctx context.Context) error {
    ctx, _ = logtracing.StartSpan(ctx, "DoWork")
    var err error
    defer func() {
        logtracing.EndSpan(ctx, err) // err is always nil!
    }()
    return errors.New("some error")
}

// ✅ Correct: Named return value
func DoWork(ctx context.Context) (xerr error) {
    ctx, _ = logtracing.StartSpan(ctx, "DoWork")
    defer func() {
        logtracing.EndSpan(ctx, xerr) // xerr captures the actual error
    }()
    return errors.New("some error")
}
```

## Troubleshooting

### No log output?

- Ensure slog default logger is set before using logtracing:

```go
slogx.SetupDefaultLogger(&slogx.Config{Level: "info"})
```

### Spans not linked?

- Ensure you're using the ctx returned by `StartSpan`

### Error not captured in defer?

- Use named return value (`xerr`)

## References

- [logtracing source](https://github.com/theplant/appkit/tree/master/logtracing)
- [slogx docs](https://pkg.go.dev/github.com/qor5/x/v3/slogx)
- [gormx docs](https://pkg.go.dev/github.com/qor5/x/v3/gormx)
- [goquex docs](https://pkg.go.dev/github.com/qor5/x/v3/goquex)
