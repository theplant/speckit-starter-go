---
description: Setup Go service with confx configuration management, lifecycle-based dependency injection, and provider pattern (with mandatory checkpoints).
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Help project setup standardized service startup architecture with:

- **Configuration management** with `confx` (embedded defaults + file + env + flags)
- **Lifecycle management** with `lifecycle` (dependency injection, graceful shutdown)
- **Provider pattern** (each component as independent provider function)

This workflow focuses on **framework and patterns**, not specific business logic or directory structures.

## Core Principles

1. **Configuration First**: All runtime configuration through `confx`, never scattered `os.Getenv()` calls
2. **Provider Independence**: Each component is a provider function with explicit dependencies
3. **Lifecycle Managed**: All long-running components managed by `lifecycle`
4. **Flexible Structure**: Directory structure adapts to project, not the other way around
5. **Minimal Dependency**: Only introduce dependencies that are actually used by other providers in the lifecycle

## Execution Steps

### Step 1: Analyze Current Project

1. **Examine existing structure**

   - Identify entry point(s) (`main.go`, `cmd/*/main.go`)
   - Find existing configuration patterns (env vars, config files, hardcoded values)
   - List components that should become providers (DB, services, servers, etc.)
   - Check for existing `confx`/`lifecycle` usage

2. **Propose directory structure**

   Based on analysis, propose a structure. Common patterns:

   | Pattern                                                       | When to Use                  |
   | ------------------------------------------------------------- | ---------------------------- |
   | Single `main.go` with inline config/providers                 | Small demos, simple services |
   | `cmd/main.go` + `pkg/app/config.go` + `pkg/app/provider.go`   | Medium services              |
   | `cmd/{service}/main.go` + `cmd/{service}/setup/` + `pkg/app/` | Multi-service projects       |

   **Important**: Each project should have a **single centralized package** (e.g., `pkg/app/` or `app/`) that provides:

   - Config structs and `InitializeXxxConfig` functions
   - Provider functions
   - Embedded default YAML configuration files

   This package serves as the single source of truth for dependency injection, whether for the project's own service startup or when other projects depend on this project's services.

   **Service Startup Principles**:

   - `cmd/` is responsible for service startup, defining a `Setup` that lists required providers
   - `main.go` creates a lifecycle, loads config from the centralized package, then invokes `Setup`
   - For **single service**: one lifecycle with providers from the centralized package
   - For **multiple services in one cmd**: define a parent lifecycle containing child lifecycles for each service. If service B depends on a value from service A after startup, use `lc.Resolve()` on service A's lifecycle to obtain it

   **ASK USER** to confirm the proposed structure before proceeding.

3. **Identify configuration items**

   Extract all configurable values:

   - Database connection strings
   - Server addresses/ports
   - External service credentials (OAuth, API keys)
   - Feature flags
   - Timeouts and limits

#### ⛔ Step 1 Checkpoint - MANDATORY

**You MUST complete the following before proceeding to Step 2:**

1. **Output your analysis** to the user:

   ```
   ## Step 1 Analysis Results

   ### Existing Structure
   - Entry point(s): [list files]
   - Existing config patterns: [describe]
   - Components to become providers: [list]
   - confx/lifecycle usage: [yes/no, describe]

   ### Proposed Directory Structure
   - Pattern: [single main.go | cmd + pkg/app | multi-service]
   - Files to create/modify: [list]

   ### Configuration Items Identified
   - [list all configurable values]
   ```

2. **If any issues found** → Generate your proposed solution and include it in the output.

3. **Continue to Step 2** immediately after outputting the analysis. Do NOT wait for user confirmation.

---

### Step 2: Create Configuration Structure

1. **Define Config struct**

   ```go
   type Config struct {
       // Infrastructure configs (use qor5/x types when available)
       Log      slogx.Config         `confx:"log"`
       Database gormx.DatabaseConfig `confx:"database"`
       HTTP     httpx.ServerConfig   `confx:"http"`

       // Business configs
       // ... project-specific configs
   }
   ```

   Rules:

   - Use `confx:"fieldName"` tag (camelCase)
   - Use `usage:"description"` tag for documentation
   - Use `validate:"required"` for mandatory fields
   - Group infrastructure configs first, then business configs

2. **Create embedded default config**

   Create `embed/default.yaml` (or similar path based on project structure):

   ```yaml
   log:
     level: info
   database:
     host: localhost
     port: 5432
     # NO passwords or production values
   http:
     addr: ":8080"
   ```

   Rules:

   - Provide complete defaults for local development
   - **NO** sensitive information (passwords, API keys)
   - **NO** production addresses
   - Service should start with defaults alone (for local dev)

3. **Implement config loader**

   ```go
   //go:embed embed/default.yaml
   var defaultConfigYAML string

   func InitializeConfig(opts ...confx.Option) (confx.Loader[*Config], error) {
       def, err := confx.Read[*Config]("yaml", strings.NewReader(defaultConfigYAML))
       if err != nil {
           return nil, errors.Wrap(err, "failed to load default config")
       }
       return confx.Initialize(def, opts...)
   }
   ```

#### ⛔ Step 2 Checkpoint - MANDATORY

**STOP. You MUST verify the following before proceeding to Step 3:**

1. **Config struct verification**:

   - [ ] All fields use `confx:"fieldName"` tag (camelCase)
   - [ ] Required fields have `validate:"required"` tag
   - [ ] Infrastructure configs grouped before business configs

2. **Default config verification**:

   - [ ] `embed/default.yaml` exists (or equivalent path)
   - [ ] NO sensitive information (passwords, API keys)
   - [ ] NO production addresses
   - [ ] Service can start with defaults alone

3. **Output to user**:

   ```
   ## Step 2 Verification

   ### Config Struct
   - File: [path]
   - Fields: [list with tags]

   ### Default Config
   - File: [path]
   - Contains sensitive info: NO ✅ / YES ❌
   - Contains production addresses: NO ✅ / YES ❌
   ```

**If any check fails → FIX before proceeding.**

---

### Step 3: Define Provider Functions

Each component should be a provider function following these rules:

1. **Naming conventions**:

   - **Single-function provider**: `Setup{Component}` - returns a single component
   - **Multi-function provider**: `Setup{Component}` - uses `[]any` slice containing multiple related providers
   - **Serving-related**: `Setup{Protocol}Serving` - combines Listener + Server

2. **Provider function signature**
   single-function provider:

   ```go
   // Dependencies as parameters, output as return value
   func SetupXXX(ctx context.Context, lc *lifecycle.Lifecycle, conf *Config, dep1 Type1, dep2 Type2) (*XXX, error)
   ```

   multiple-function provider:
   // use []any slice

   ```go
   var SetupXXX = []any{
    func(conf *Config) *XXX { return &conf.XXX },
    SetupXXX,
   }
   ```

   Serving combination:

   ```go
   // Serving combination
   var SetupXXXServing = []any{
       SetupXXXListener,
       SetupXXXServer,
   }
   ```

   Internal service provider (registers service to existing server):

   ```go
   var SetupLedgerServices = []any{
       // Extract service config from main config
       func(conf *Config) *ledger.Config {
           return &conf.Ledger
       },
       // Construct internal service and register to server
       func(lc *lifecycle.Lifecycle, config *ledger.Config, grpcServer *grpc.Server) (*ledger.LedgerService, error) {
           service, err := ledger.NewService(config)
           if err != nil {
               return nil, err
           }
           ledgerv1.RegisterLedgerServiceServer(grpcServer, service)
           return service, nil
       },
   }
   ```

   - **Input**: Dependencies declared via function parameters
   - **Output**: Concrete type (NOT `any`), with optional `error`
   - Accept `*lifecycle.Lifecycle` if needs lifecycle hooks
   - Accept `context.Context` as first param if needed

3. **Provider categories**

   | Category              | Examples                    | Notes                               |
   | --------------------- | --------------------------- | ----------------------------------- |
   | **Infrastructure**    | Logger, ErrorNotifier       | Usually no lifecycle hooks          |
   | **Data Layer**        | Database, Cache, MessageBus | Register cleanup in lifecycle       |
   | **External Clients**  | gRPC clients, HTTP clients  | May need connection management      |
   | **Business Services** | Domain services             | Depend on data layer                |
   | **Servers**           | HTTP server, gRPC server    | See gRPC/HTTP Serving Rules below   |
   | **One-time Tasks**    | Migrator, Seeder            | Return marker type after completion |

#### gRPC Serving Rule

gRPC serving setup follows a standard pattern with listener, server, and interceptor chain:

```go
// SetupGRPCServing combines listener and server setup as a provider group
var SetupGRPCServing = []any{
    SetupGRPCListener,
    SetupGRPCServer,
}

// SetupGRPCListener creates a TCP listener for gRPC server
// The listener is managed by lifecycle for graceful shutdown
func SetupGRPCListener(lc *lifecycle.Lifecycle, conf *Config) (grpcx.Listener, error) {
    return grpcx.SetupListener(lc, &conf.GRPC)
}

// SetupGRPCConn creates a gRPC client connection to the local gRPC server
// This is typically used for:
// - HTTP health check middleware to verify gRPC server health
// - HTTP-to-gRPC proxying scenarios
// Note: This connects to the same process's gRPC server via the listener address
func SetupGRPCConn(lc *lifecycle.Lifecycle, grpcListener grpcx.Listener) (*grpc.ClientConn, error) {
    return grpcx.SetupConnFactory("grpc-conn")(lc, &grpcx.ConnConfig{
        Address:             grpcListener.Addr().String(),
        LoadBalancingPolicy: grpcx.LoadBalancingPolicyPickFirst,
    })
}

// SetupGRPCServer creates and configures the gRPC server with interceptor chain
// Dependencies:
// - listener: TCP listener for accepting connections
// - ib: i18n bundle for error message localization
// - errorNotifier: for reporting errors to external services (e.g., Sentry)
func SetupGRPCServer(
    ctx context.Context,
    lc *lifecycle.Lifecycle,
    listener grpcx.Listener,
    ib *i18nx.I18N,
    errorNotifier errornotifier.Notifier,
    conf *Config,
) (*grpc.Server, error) {
    // Setup health service for Kubernetes probes and load balancer health checks
    healthSvc := health.NewServer()
    healthSvc.SetServingStatus("", grpc_health_v1.HealthCheckResponse_SERVING)

    server, err := grpcx.SetupServerFactory("grpc-server", grpc.ChainUnaryInterceptor(
        // Health check interceptor: allows health check requests to bypass other interceptors
        healthz.UnaryServerInterceptor(healthSvc),
        // Business interceptor chain (see grpcServerInterceptorFactory for details)
        grpcServerInterceptorFactory(ib, errorNotifier),
    ))(ctx, lc, listener, &conf.GRPC)
    if err != nil {
        return nil, err
    }

    // Register health service for gRPC health check protocol
    grpc_health_v1.RegisterHealthServer(server, healthSvc)
    return server, nil
}

// grpcServerInterceptorFactory creates the business interceptor chain
// The chain is executed in order: first interceptor wraps the outermost layer
//
// Request flow (outside → inside):
//   errInter(outer) → normalize → logtracing → statusx → errInter(inner) → handler
//
// Response flow (inside → outside):
//   handler → errInter(inner) → statusx → logtracing → normalize → errInter(outer)
func grpcServerInterceptorFactory(ib *i18nx.I18N, notifier errornotifier.Notifier) grpc.UnaryServerInterceptor {
    // Error handling interceptor (used at both outer and inner layers)
    // - DefaultErrorUnaryServerInterceptor: reports errors to notifier (e.g., Sentry)
    // - RecoveryUnaryServerInterceptor: recovers from panics and converts to gRPC error
    errInter := grpcx.ChainUnaryServerInterceptor(
        grpcx.DefaultErrorUnaryServerInterceptor(notifier),
        grpcx.RecoveryUnaryServerInterceptor(),
    )

    inters := []grpc.UnaryServerInterceptor{
        // [Outer error handler]: Catches errors from all inner interceptors,
        // reports to error notifier, and ensures panic recovery at outermost layer
        errInter,

        // [Normalize]: Normalizes request data (e.g., trim whitespace, normalize unicode)
        // Should run early to ensure all downstream handlers see clean data
        normalize.GRPCUnaryServerInterceptor(),

        // [Log & Tracing]: Adds request logging and distributed tracing
        // Creates trace context and logs request/response metadata
        logtracing.UnaryServerInterceptor(),

        // [Status translation]: Converts internal errors to user-friendly gRPC status
        // Uses i18n bundle to localize error messages based on client's Accept-Language
        statusx.UnaryServerInterceptor(ib),

        // [Inner error handler]: Catches errors from business handler,
        // ensures errors are properly wrapped before status translation
        errInter,
    }
    return grpcx.ChainUnaryServerInterceptor(inters...)
}
```

**Key points:**

- Use `grpcx.SetupListener` and `grpcx.SetupServerFactory` for lifecycle management
- Register health service for Kubernetes probes and load balancer health checks
- **Interceptor chain design**: error handling wraps both outer and inner layers to ensure:
  - Outer layer: catches errors from interceptors, final panic recovery
  - Inner layer: catches business errors before status translation
- `SetupGRPCConn` creates a client connection to the local gRPC server (for HTTP health check)

#### HTTP Serving Rule

HTTP serving setup includes mux, listener, server, and protobuf-to-HTTP handler:

```go
// SetupHTTPServing combines all HTTP serving components as a provider group
// Note: SetupGRPCConn is included because HTTP health check needs to verify gRPC server health
var SetupHTTPServing = []any{
    SetupHTTPServeMux,
    SetupHTTPListener,
    SetupProttpHandler,
    SetupHTTPServer,
    SetupGRPCConn,
}

// SetupHTTPListener creates a TCP listener for HTTP server
// The listener is managed by lifecycle for graceful shutdown
func SetupHTTPListener(lc *lifecycle.Lifecycle, conf *Config) (httpx.Listener, error) {
    return httpx.SetupListener(lc, &conf.HTTP)
}

// SetupHTTPServeMux creates a new HTTP request multiplexer
// This is the base router where all HTTP handlers are registered
func SetupHTTPServeMux() *http.ServeMux {
    return http.NewServeMux()
}

// SetupHTTPServer creates and configures the HTTP server with middleware chain
// Dependencies:
// - listener: TCP listener for accepting connections
// - conn: gRPC client connection for health check middleware
// - mux: HTTP request multiplexer
// - prottpHandler: protobuf-to-HTTP bridge handler
func SetupHTTPServer(
    ctx context.Context,
    lc *lifecycle.Lifecycle,
    listener httpx.Listener,
    conf *Config,
    ib *i18nx.I18N,
    conn *grpc.ClientConn,
    mux *http.ServeMux,
    prottpHandler *prottpx.Handler,
) (http.Handler, *http.Server, error) {
    // Register prottpHandler as the default handler for all routes
    // prottpHandler bridges HTTP requests to gRPC-style service handlers
    mux.Handle("/", prottpHandler)

    // Merge CORS allowed headers from all packages that add custom headers
    // This ensures browsers can access these headers in cross-origin requests
    // Each package exports its required headers (e.g., i18nx.AllowHeaders for Accept-Language)
    conf.HTTP.Security.CORS.AllowedHeaders = lo.Uniq(
        slices.Concat(
            conf.HTTP.Security.CORS.AllowedHeaders,
            i18nx.AllowHeaders,    // i18n headers (Accept-Language, etc.)
            statusx.AllowHeaders,  // status headers for error details
            auth.AllowHeaders,     // auth headers (Authorization, etc.)
            challenge.AllowHeaders, // challenge headers for MFA/verification
        ),
    )

    // HTTP middleware chain - executed from top to bottom for requests,
    // bottom to top for responses
    //
    // Request flow (outside → inside):
    //   DefaultMiddleware → healthz → NoStore → Security → Cookieize → normalize → handler
    handler := hook.Chain(
        // [Default middleware]: Adds request ID, logging, panic recovery
        // Provides basic observability and error handling for all requests
        server.DefaultMiddleware(kitlog.Default()),

        // [Health check]: Handles /healthz endpoint, checks gRPC server health via conn
        // Returns 200 if healthy, 503 if unhealthy - used by Kubernetes probes
        healthz.HTTPMiddleware(healthz.WithGRPCHealthChecker(conn)),

        // [No-Store]: Sets Cache-Control: no-store header
        // Prevents browsers and proxies from caching sensitive API responses
        httpx.NoStore,

        // [Security]: Applies security headers and CORS policy
        // - CORS: handles preflight requests, validates origins, sets allowed headers
        // - Security headers: X-Content-Type-Options, X-Frame-Options, etc.
        httpx.Security(conf.HTTP.Security),

        // [Cookie auth]: Extracts auth token from cookie and sets to Authorization header
        // Enables cookie-based auth for browser clients while keeping token-based auth internally
        // Note: This is project-specific, remove if not using cookie-based auth
        auth.Cookieize(conf.Auth.Cookie),

        // [Normalize]: Normalizes request data (e.g., trim whitespace, normalize unicode)
        // Should run last before handler to ensure clean data for business logic
        normalize.HTTPMiddleware,
    )(mux)

    server, err := httpx.SetupServerFactory("http-server", handler)(ctx, lc, &conf.HTTP, listener)
    if err != nil {
        return nil, nil, err
    }

    return handler, server, nil
}

// SetupProttpHandler creates a protobuf-to-HTTP bridge handler
// This allows HTTP clients to call gRPC-style services using JSON/protobuf over HTTP
//
// The handler applies gRPC interceptors to HTTP requests, providing:
// - Same error handling and logging as gRPC
// - Authentication via JWT and session validation
//
// Dependencies:
// - i18n: for error message localization
// - notifier: for error reporting
// - jwt: for token validation
// - authClient: for session fetching from auth service
func SetupProttpHandler(i18n *i18nx.I18N, notifier errornotifier.Notifier, jwt *auth.JWT, authClient authv1.AuthServiceClient) *prottpx.Handler {
    return prottpx.NewHandler(
        prottpx.ChainUnaryInterceptor(
            // Reuse the same gRPC interceptor chain for consistent error handling
            // See grpcServerInterceptorFactory for interceptor details
            grpcServerInterceptorFactory(i18n, notifier),

            // [Auth interceptor]: Validates JWT token and fetches user session
            // - JWT: validates token signature and expiration
            // - SessionFetcher: retrieves full session from auth service
            // - SessionValidator: custom validation logic (e.g., check session status)
            // Note: This is project-specific, adjust based on your auth requirements
            auth.UnaryServerInterceptor(&auth.InterceptorConfig{
                JWT:            jwt,
                SessionFetcher: auth.NewSessionFetcherWithClient(authClient),
                SessionValidator: func(_ context.Context, session *auth.Session) error {
                    return errors.WithStack(session.Validate())
                },
            }),
        ),
    )
}
```

**Key points:**

- Use `httpx.SetupListener` and `httpx.SetupServerFactory` for lifecycle management
- `prottpx.Handler` bridges HTTP requests to gRPC-style handlers, enabling JSON/protobuf over HTTP
- **HTTP middleware chain** (executed top to bottom):
  - `DefaultMiddleware`: request ID, logging, panic recovery
  - `healthz`: health check endpoint for Kubernetes probes
  - `NoStore`: prevents caching of API responses
  - `Security`: CORS and security headers
  - `Cookieize`: cookie-to-header auth conversion (project-specific)
  - `normalize`: request data normalization
- **CORS headers**: merge required headers from all packages to ensure browser compatibility
- `SetupGRPCConn` is included for health check middleware to verify gRPC server health

4. **One-time task provider pattern** (Migrate)

   ```go
   type Migrator struct{}

   func Migrate(ctx context.Context, db *gorm.DB) (*Migrator, error) {
       if err := db.AutoMigrate(AllModels()...); err != nil {
           return nil, errors.Wrap(err, "failed to migrate")
       }
       return &Migrator{}, nil
   }
   ```

- Return a marker type (e.g., `*Migrator`) so other providers can depend on it
- Use dependency to enforce execution order (e.g., Seeder depends on Migrator)

5. **Worker Service Rule**

   For background workers and message consumers that need lifecycle management:

   ```go
   var SetupWorkerService = []any{
       // Extract worker-specific config from main config
       func(conf *Config) *WorkerConfig {
           return &conf.Worker
       },
       // Create worker and register to lifecycle
       func(lc *lifecycle.Lifecycle, conf *WorkerConfig, db *gorm.DB, bus bus.Bus, conn ExternalConn) (*Worker, error) {
           // Create gRPC client inside provider when this is the terminal consumer
           client := externalv1.NewExternalServiceClient(conn)

           worker, err := NewWorker(conf, db, bus, client)
           if err != nil {
               return nil, err
           }

           // Register worker using goquex.NewWorkerService
           // For workers with Start(ctx) (WorkerController, error) signature
           lc.Add(goquex.NewWorkerService(worker.Start).WithName("worker-name"))
           return worker, nil
       },
   }
   ```

   **Key points:**

   - Use `[]any` slice to group config extraction and worker creation
   - Create gRPC client inside provider when it's the **terminal consumer**
   - Use `goquex.NewWorkerService` for workers that return `WorkerController`
   - Use `.WithName()` to identify the worker in logs

6. **gRPC Connection vs Client rule**

   When a provider needs to call an external gRPC service:

   - **Define typed `Conn` for each external service**
   - **Create `SetupXxxClient` provider ONLY when the client is shared by multiple providers**
   - **Create client inside provider when it's the terminal consumer**

   ✅ **Pattern A: Shared client (multiple consumers)**

   When multiple providers need the same client:

   ```go
   // Define typed connection
   type ConsentConn grpc.ClientConnInterface

   func SetupConsentConn(lc *lifecycle.Lifecycle, conf *Config) (ConsentConn, error) {
       return grpcx.SetupConnFactory("consent-conn")(lc, &conf.Consent)
   }

   // Create shared client provider - multiple providers depend on this
   func SetupConsentServiceClient(conn ConsentConn) consentv1.ConsentServiceClient {
       return consentv1.NewConsentServiceClient(conn)
   }

   // Multiple providers depend on the shared client
   func SetupServiceA(client consentv1.ConsentServiceClient) *ServiceA { ... }
   func SetupServiceB(client consentv1.ConsentServiceClient) *ServiceB { ... }
   ```

   ✅ **Pattern B: Terminal consumer (single consumer)**

   When only one provider needs the client, create it inside:

   ```go
   var SetupConsumerService = []any{
       func(conf *Config) *ConsumerConfig { return &conf.Consumer },
       func(lc *lifecycle.Lifecycle, conf *ConsumerConfig, conn ExternalConn) (*Consumer, error) {
           // Create client inside - this is the only consumer
           client := externalv1.NewExternalServiceClient(conn)
           consumer, err := NewConsumer(conf, client)
           // ...
       },
   }
   ```

   ❌ **Anti-pattern: Creating unused shared client**

   ```go
   // DON'T create SetupXxxClient if only one provider uses it
   func SetupExternalClient(conn ExternalConn) externalv1.Client { ... }

   // Only one consumer - should create client inside instead
   func SetupConsumer(client externalv1.Client) *Consumer { ... }
   ```

7. **Verify provider dependencies (Minimal Dependency Principle)**

   After creating a provider, verify that:

   - **Every provider's return type is used by at least one other provider in the lifecycle**, OR
   - **The provider registers lifecycle hooks** (OnStart/OnStop), OR
   - **The provider performs a one-time task** (like migration)

   If a provider's return type is not depended upon by any other provider and it doesn't register lifecycle hooks, it should NOT be added to the provider list.

   **Verification checklist:**

   ```
   For each new provider SetupXXX that returns *XXX:
   1. Search the codebase: Is *XXX used as a parameter in any other provider?
   2. Does SetupXXX call lc.Add() or lc.Append() to register lifecycle hooks?
   3. Is SetupXXX a one-time task (migration, seeding)?

   If ALL answers are NO → Do NOT add this provider to the lifecycle
   ```

   ✅ **Valid reasons to add a provider:**

   - Its return type is a dependency of another provider
   - It registers lifecycle hooks (starts a server, worker, etc.)
   - It performs a one-time initialization task

   ❌ **Invalid reason:**

   - "It might be useful later" - violates minimal dependency principle
   - "Other services might need it" - add it when actually needed

#### ⛔ Step 3 Checkpoint - MANDATORY

**STOP. You MUST complete the following verification for EACH provider before proceeding to Step 4:**

```
## Step 3 Provider Verification

For each provider, verify ALL applicable rules:

| Provider | Returns | Format | gRPC Client | Lifecycle |
|----------|---------|--------|-------------|-----------|
| (name)   | (type)  | (A/B)  | (C/D/N/A)   | (E/F/N/A) |

### Format Check:
- **A**: Uses `var SetupXXX = []any{...}` for multi-function providers ✅
- **B**: Uses `func SetupXXX(...) (Type, error)` for single-function providers ✅

### gRPC Client Check (if calls external gRPC):
- **C**: Shared client - has `SetupXxxClient` provider, multiple consumers depend on it ✅
- **D**: Terminal consumer - creates client inside provider, no other consumer needs it ✅

### Lifecycle Check (if needs lifecycle management):
- **E**: Uses `lc.Add(goquex.NewWorkerService(...))` for workers ✅
- **F**: Uses `lc.Add()` with custom Actor/Service for other cases ✅

### Minimal Dependency Check:
- [ ] Return type is used by another provider, OR
- [ ] Registers lifecycle hooks via `lc.Add()`, OR
- [ ] Performs one-time task (migration, seeding)
```

**If ANY check fails → FIX before proceeding.**

---

### Step 4: Assemble Providers

1. **Create setup variable**

   **IMPORTANT**: Setup MUST be a `var` of type `[]any`, NOT a function.

   ```go
   package setup

   import "github.com/yourproject/pkg/app"

   // Setup is a slice of providers - NOT a function
   var Setup = []any{
       // 1. Infrastructure
       app.SetupI18N,
       app.SetupLogger,
       app.SetupErrorNotifier,

       // 2. External connections
       app.SetupCIAMConn,
       app.SetupConsentConn,
       app.SetupConsentServiceClient,  // Shared client (multiple consumers)

       // 3. Servers
       app.SetupGRPCServing,
       app.SetupHTTPServing,

       // 4. Data layer
       app.SetupDatabase,
       app.SetupGoBus,

       // 5. Business services
       app.SetupLoyaltyServices,
       app.SetupLoyaltyServiceConn,
       app.SetupLoyaltyServiceClient,  // Shared client

       // 6. Workers
       app.SetupConsumerService,  // Terminal consumer, creates client inside
   }
   ```

   ❌ **Incorrect pattern:**

   ```go
   // DON'T use function that calls lc.Provide manually
   func Setup(lc *lifecycle.Lifecycle, conf *Config) {
       lc.Provide(...)
   }
   ```

2. **Provider ordering rules**
   - Dependencies must come before dependents
   - Group by category for readability
   - One-time tasks before services that depend on migrated schema

#### ⛔ Step 4 Checkpoint - MANDATORY

**STOP. You MUST verify provider ordering and usage before proceeding to Step 5:**

1. **Dependency order check**: For each provider, verify all its dependencies appear earlier in the list.

2. **Unused provider check**: Verify all provider functions defined in the package are included in the Setup slice. There should be NO unused provider functions.

3. **Output to user**:

   ```
   ## Step 4 Provider Order & Usage Verification

   | # | Provider | Dependencies | All deps appear earlier? | Included in Setup? |
   |---|----------|--------------|-------------------------|--------------------|
   | 1 | (name)   | (list)       | ✅/❌                    | ✅/❌               |
   | 2 | (name)   | (list)       | ✅/❌                    | ✅/❌               |
   ...

   ### Unused Provider Functions Check
   - All `Setup*` functions in package are used: ✅/❌
   - Unused functions found: [list or "None"]
   ```

**If ANY row has ❌ → Fix before proceeding:**

- Dependency order issue → Reorder providers
- Unused provider → Either add to Setup or remove the function

---

### Step 5: Create Entry Point

**MUST** use `github.com/spf13/pflag` for command-line flag parsing.

1. **Standard main.go pattern**

   ```go
   package main

   import (
       "context"
       "log"
       "os"

       "github.com/qor5/confx"
       "github.com/spf13/pflag"
       "github.com/theplant/inject/lifecycle"

       "github.com/yourproject/cmd/myservice/setup"
       "github.com/yourproject/pkg/app"
   )

   const envPrefix = "MYAPP_"

   func main() {
       var confPath string
       flagSet := pflag.NewFlagSet(os.Args[0], pflag.ContinueOnError)
       flagSet.SortFlags = false
       flagSet.StringVarP(&confPath, "config", "c", "", "Path to config yaml file")

       confLoader, err := app.InitializeConfig(
           confx.WithEnvPrefix(envPrefix),
           confx.WithFlagSet(flagSet),
       )
       if err != nil {
           log.Fatalf("Failed to initialize config: %v", err)
       }

       providers := []any{
           lifecycle.SetupSignal,
           func(ctx context.Context) (*app.Config, error) {
               return confLoader(ctx, confPath)
           },
           setup.Setup,
       }

       if err := lifecycle.Serve(context.Background(), providers...); err != nil {
           log.Fatalf("Failed to serve: %v", err)
       }
   }
   ```

2. **Separate command pattern** (e.g., migrate-only)

   ```go
   // cmd/migrate/main.go
   func main() {
       // ... config setup same as above

       providers := []any{
           func(ctx context.Context) (*Config, error) {
               return confLoader(ctx, confPath)
           },
           SetupDatabase,
           Migrate,
       }

       lc := lifecycle.New()
       if err := lc.Provide(providers...); err != nil {
           log.Fatalf("Failed to provide: %v", err)
       }
       if err := lc.Start(context.Background()); err != nil {
           log.Fatalf("Failed to start: %v", err)
       }
       // Migrate completed, exit
   }
   ```

#### ⛔ Step 5 Checkpoint - MANDATORY

**STOP. You MUST verify the entry point before proceeding to Step 6:**

1. **main.go verification**:

   - [ ] Uses `github.com/spf13/pflag` for flag parsing
   - [ ] Does NOT call `pflag.Parse()` manually
   - [ ] Includes `lifecycle.SetupSignal` for graceful shutdown
   - [ ] Config loaded via `confLoader(ctx, confPath)`
   - [ ] No business logic in main.go

2. **Output to user**:

   ```
   ## Step 5 Entry Point Verification

   - File: [path]
   - Uses pflag: ✅/❌
   - Manual pflag.Parse(): NO ✅ / YES ❌
   - Has lifecycle.SetupSignal: ✅/❌
   - Business logic in main: NO ✅ / YES ❌
   ```

**If ANY check fails → FIX before proceeding.**

---

### Step 6: Verify Setup

1. **Build check**
   // turbo

   ```bash
   go build ./...
   ```

2. **Run with defaults**

   ```bash
   ./cmd/myapp/myapp
   ```

3. **Run with config file**

   ```bash
   ./cmd/myapp/myapp -c config.yaml
   ```

#### ⛔ Step 6 Checkpoint - MANDATORY

**You MUST verify the build succeeds AND the service starts correctly:**

1. **Run build check**:
   // turbo

   ```bash
   go build ./...
   ```

2. **If build fails** → Fix the errors immediately, then re-run build check.

3. **Run service startup check**:

   After successful build, start the service to verify it initializes correctly:

   ```bash
   # Run the service in background, wait for startup, then check if it's running
   timeout 10s ./cmd/myapp/myapp &
   sleep 3
   # Check if process is still running (not crashed)
   # If using HTTP server, can also curl health endpoint:
   # curl -f http://localhost:8080/healthz || echo "Health check failed"
   ```

   **Verification criteria:**

   - Service starts without immediate crash
   - No panic or fatal errors in startup logs
   - If HTTP server: health endpoint responds
   - If gRPC server: can establish connection

4. **If startup fails** → Analyze error logs, fix the issue, rebuild and re-test.

5. **Output to user**:

   ```
   ## Step 6 Final Verification

   - Build result: SUCCESS ✅ / FAILED ❌
   - Service startup: SUCCESS ✅ / FAILED ❌
   - Startup logs: [brief summary or "Clean startup"]
   - If failed initially, fixes applied: [describe fixes]
   ```

6. **Continue** to mark task complete after successful build AND startup. Do NOT wait for user confirmation.

---

## Configuration Priority (Low to High)

1. Embedded defaults (`embed/default.yaml`)
2. Config file (`-c/--config`)
3. Environment variables (`{PREFIX}_FIELD_NAME`)
4. Command-line flags

## AI Agent Requirements

### MUST DO

- Analyze existing project structure before proposing changes
- **ASK USER** to confirm directory structure proposal
- Use `confx` for all configuration (no scattered `os.Getenv()`)
- Each component as independent provider function
- Use `lifecycle` for dependency injection and lifecycle management
- Provider functions return concrete types with explicit dependencies
- One-time tasks (migrate/seed) return marker types for dependency ordering
- Include `lifecycle.SetupSignal` for graceful shutdown
- **Setup must be `var Setup = []any{...}`** - not a function
- **Use `goquex.NewWorkerService` for workers** that return `WorkerController`
- **Verify minimal dependency principle** - only add providers that are actually used

### MUST NOT

- Assume fixed directory structure without analysis
- Use global variables in providers
- Include business logic in main.go
- Call `pflag.Parse()` manually (confx handles it)
- Hardcode configuration values that should be configurable
- **Use `func Setup(lc, conf)` pattern** - Setup must be `var Setup = []any{...}`
- **Create custom Service wrapper types** when `goquex.NewWorkerService` or similar utilities exist
- **Create `SetupXxxClient` providers for terminal consumers** - create client inside provider instead
- **Add providers whose return type is not depended upon** - unless they register lifecycle hooks or perform one-time tasks

## Common Infrastructure Packages

### Core Packages

| Package     | Import                                 | Purpose                                       |
| ----------- | -------------------------------------- | --------------------------------------------- |
| `confx`     | `github.com/qor5/confx`                | Configuration management (YAML/env/flags)     |
| `lifecycle` | `github.com/theplant/inject/lifecycle` | Dependency injection and lifecycle management |
| `pflag`     | `github.com/spf13/pflag`               | Command-line flag parsing                     |

### Server & Network Packages

| Package   | Import                         | Purpose                                           |
| --------- | ------------------------------ | ------------------------------------------------- |
| `httpx`   | `github.com/qor5/x/v3/httpx`   | HTTP server setup and middleware                  |
| `grpcx`   | `github.com/qor5/x/v3/grpcx`   | gRPC server/client connection and interceptors    |
| `prottpx` | `github.com/qor5/x/v3/prottpx` | Protobuf-to-HTTP bridge (JSON/protobuf over HTTP) |
| `healthz` | `github.com/qor5/x/v3/healthz` | Health check (HTTP middleware & gRPC interceptor) |

### Data & Messaging Packages

| Package  | Import                        | Purpose                            |
| -------- | ----------------------------- | ---------------------------------- |
| `gormx`  | `github.com/qor5/x/v3/gormx`  | Database connection (GORM wrapper) |
| `gobusx` | `github.com/qor5/x/v3/gobusx` | Message bus setup                  |
| `goquex` | `github.com/qor5/x/v3/goquex` | Worker service (background jobs)   |

### Observability & Error Handling Packages

| Package          | Import                                  | Purpose                                 |
| ---------------- | --------------------------------------- | --------------------------------------- |
| `slogx`          | `github.com/qor5/x/v3/slogx`            | Structured logging (slog wrapper)       |
| `errornotifierx` | `github.com/qor5/x/v3/errornotifierx`   | Error notification (e.g., Sentry)       |
| `logtracing`     | `github.com/theplant/appkit/logtracing` | Request logging and distributed tracing |

### Request Processing Packages

| Package     | Import                           | Purpose                                    |
| ----------- | -------------------------------- | ------------------------------------------ |
| `normalize` | `github.com/qor5/x/v3/normalize` | Request data normalization (trim, unicode) |
| `statusx`   | `github.com/qor5/x/v3/statusx`   | gRPC status translation with i18n          |
| `i18nx`     | `github.com/qor5/x/v3/i18nx`     | Internationalization support               |
| `hook`      | `github.com/qor5/x/v3/hook`      | Middleware chaining utility                |

### Utility Packages

| Package  | Import                  | Purpose                                |
| -------- | ----------------------- | -------------------------------------- |
| `errors` | `github.com/pkg/errors` | Error wrapping with stack traces       |
| `lo`     | `github.com/samber/lo`  | Generic utility functions (slice, map) |
