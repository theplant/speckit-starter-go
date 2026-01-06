---
description: gRPC and HTTP serving setup workflow for Go services.
---

# Serving Setup Workflow

This document provides a 6-step workflow to help projects quickly integrate HTTP and gRPC services.

---

## Step 1: Confirm Requirements

Before starting, confirm the following three questions:

| Question       | Options        | Description                                   |
| -------------- | -------------- | --------------------------------------------- |
| **Need gRPC?** | ✅ Yes / ❌ No | Internal service communication, microservices |
| **Need HTTP?** | ✅ Yes / ❌ No | External API, Web clients                     |
| **Need Auth?** | ✅ Yes / ❌ No | User authentication required                  |

**Common Combinations:**

| Scenario                     | gRPC | HTTP | Auth  | Description                                   |
| ---------------------------- | ---- | ---- | ----- | --------------------------------------------- |
| Full Microservice (Public)   | ✅   | ✅   | ✅    | Internal gRPC + External HTTP API + User Auth |
| Full Microservice (Internal) | ✅   | ✅   | ❌    | Internal gRPC + External HTTP API, No Auth    |
| HTTP Only Service            | ❌   | ✅   | ✅/❌ | Simple Web service, optional Auth             |
| Internal Only Service        | ✅   | ❌   | ❌    | Only called by other services                 |

---

## Step 2: Analyze Code Structure

Determine where the provider list should be placed.

### Recommended Directory Structure

```
pkg/
└── app/
    ├── config.go      # Config struct definition
    ├── provider.go    # All Setup* provider functions
    └── migrate.go     # Database migration (if needed)

cmd/
└── myservice/
    └── main.go        # Combine providers, start service
```

### Basic Config Structure

```go
// pkg/app/config.go
package app

import (
    "github.com/qor5/x/v3/errornotifierx"
    "github.com/qor5/x/v3/grpcx"
    "github.com/qor5/x/v3/httpx"
    "github.com/qor5/x/v3/slogx"
)

type Config struct {
    // Basic config (required)
    Log           slogx.Config           `yaml:"log"`
    ErrorNotifier *errornotifierx.Config `yaml:"error_notifier"`

    // Serving config (choose based on requirements)
    GRPC grpcx.Config `yaml:"grpc"` // Step 3: Required for gRPC
    HTTP httpx.Config `yaml:"http"` // Step 4: Required for HTTP

    // Auth config (Step 5: Required for Auth)
    // Auth AuthConfig       `yaml:"auth"`
    // CIAM grpcx.ConnConfig `yaml:"ciam"`
}
```

### Basic Providers (Required)

```go
// pkg/app/provider.go
package app

var SetupLogger = []any{
    func(conf *Config) *slogx.Config { return &conf.Log },
    slogx.SetupDefaultLogger,
}

var SetupErrorNotifier = []any{
    func(conf *Config) *errornotifierx.Config { return conf.ErrorNotifier },
    errornotifierx.SetupNotifier,
}

func SetupI18N() (*i18nx.I18N, error) {
    return i18n.New() // Use project's i18n package
}
```

### main.go Template

```go
// cmd/myservice/main.go
package main

func main() {
    providers := []any{
        // Infrastructure (required)
        app.SetupLogger,
        app.SetupErrorNotifier,
        app.SetupI18N,

        // Step 3: gRPC (if needed)
        // app.SetupGRPCServing,

        // Step 4: HTTP (if needed)
        // app.SetupHTTPServing,

        // Step 5: Auth (if needed)
        // app.SetupAuthProviders,

        // Business services
        // app.SetupMyService,
    }

    // Start using inject
    // ...
}
```

### Core Dependencies

This workflow uses the following core dependencies:

| Step   | Package                         | Description                                                 |
| ------ | ------------------------------- | ----------------------------------------------------------- |
| Basic  | `github.com/qor5/x/v3`          | Provides grpcx, httpx, healthz, slogx, errornotifierx, etc. |
| Basic  | `github.com/theplant/appkit`    | Provides server.DefaultMiddleware                           |
| Basic  | `github.com/theplant/inject`    | Dependency injection                                        |
| Step 3 | `google.golang.org/grpc`        | gRPC core library                                           |
| Step 5 | `github.com/theplant/ciam-next` | Auth authentication                                         |

**Resolving Dependency Conflicts:**

If you encounter `ambiguous import` errors during `go mod tidy` (e.g., `cloud.google.com/go/compute/metadata`), run:

```bash
go get cloud.google.com/go/compute/metadata@latest
go mod tidy
```

---

## Step 3: Integrate gRPC

> Skip this step if gRPC is not needed.

### 3.1 Add Configuration

```go
// pkg/app/config.go
type Config struct {
    // ... basic config
    GRPC grpcx.Config `yaml:"grpc"`
}
```

```yaml
# config.yaml
grpc:
  port: 9090
```

### 3.2 Add Provider

```go
// pkg/app/provider.go

var SetupGRPCServing = []any{
    SetupGRPCListener,
    SetupGRPCServer,
}

func SetupGRPCListener(lc *lifecycle.Lifecycle, conf *Config) (grpcx.Listener, error) {
    return grpcx.SetupListener(lc, &conf.GRPC)
}

func SetupGRPCServer(
    ctx context.Context,
    lc *lifecycle.Lifecycle,
    listener grpcx.Listener,
    ib *i18nx.I18N,
    errorNotifier errornotifier.Notifier,
    conf *Config,
) (*grpc.Server, error) {
    healthSvc := health.NewServer()
    healthSvc.SetServingStatus("", grpc_health_v1.HealthCheckResponse_SERVING)

    server, err := grpcx.SetupServerFactory("grpc-server", grpc.ChainUnaryInterceptor(
        healthz.UnaryServerInterceptor(healthSvc),
        grpcServerInterceptorFactory(ib, errorNotifier),
    ))(ctx, lc, listener, &conf.GRPC)
    if err != nil {
        return nil, err
    }

    grpc_health_v1.RegisterHealthServer(server, healthSvc)
    return server, nil
}

// grpcServerInterceptorFactory creates gRPC interceptor chain
func grpcServerInterceptorFactory(ib *i18nx.I18N, notifier errornotifier.Notifier) grpc.UnaryServerInterceptor {
    errInter := grpcx.ChainUnaryServerInterceptor(
        grpcx.DefaultErrorUnaryServerInterceptor(notifier),
        grpcx.RecoveryUnaryServerInterceptor(),
    )
    return grpcx.ChainUnaryServerInterceptor(
        errInter,
        normalize.GRPCUnaryServerInterceptor(),
        logtracing.UnaryServerInterceptor(),
        statusx.UnaryServerInterceptor(ib),
        errInter,
    )
}
```

### 3.3 Register Business Service

```go
// pkg/app/provider.go

func SetupMyService(db *gorm.DB, grpcServer *grpc.Server) *MyService {
    service := NewMyService(db)
    myv1.RegisterMyServiceServer(grpcServer, service)
    return service
}
```

### 3.4 Enable gRPC

```go
// cmd/myservice/main.go
providers := []any{
    // ...
    app.SetupGRPCServing,  // Add this line
    app.SetupMyService,
}
```

### 3.5 Checkpoint

After completing Step 3, verify the following items:

| #   | Check Item                                                         | Required | Verification Method                                                                          |
| --- | ------------------------------------------------------------------ | -------- | -------------------------------------------------------------------------------------------- |
| 1   | Register health check with `healthz.UnaryServerInterceptor`        | ✅       | Confirm `SetupGRPCServer` calls `healthz.UnaryServerInterceptor(healthSvc)`                  |
| 2   | Register health service with `grpc_health_v1.RegisterHealthServer` | ✅       | Confirm `SetupGRPCServer` ends with `grpc_health_v1.RegisterHealthServer(server, healthSvc)` |
| 3   | Error reporting with `grpcx.DefaultErrorUnaryServerInterceptor`    | ✅       | Confirm `grpcServerInterceptorFactory` includes this interceptor                             |
| 4   | Panic recovery with `grpcx.RecoveryUnaryServerInterceptor`         | ✅       | Confirm `grpcServerInterceptorFactory` includes this interceptor                             |
| 5   | Log tracing with `logtracing.UnaryServerInterceptor`               | ✅       | Confirm `grpcServerInterceptorFactory` includes this interceptor                             |
| 6   | Error status i18n with `statusx.UnaryServerInterceptor`            | ✅       | Confirm `grpcServerInterceptorFactory` includes this interceptor with `i18nx.I18N`           |
| 7   | Data normalization with `normalize.GRPCUnaryServerInterceptor`     | ✅       | Confirm `grpcServerInterceptorFactory` includes this interceptor                             |
| 8   | `errInter` called at both start and end of interceptor chain       | ✅       | Confirm chain is `errInter → normalize → logtracing → statusx → errInter`                    |

**If check fails:** Refer to section 3.2 code to fix `SetupGRPCServer` and `grpcServerInterceptorFactory`.

---

## Step 4: Integrate HTTP

> Skip this step if HTTP is not needed.

### 4.1 Add Configuration

```go
// pkg/app/config.go
type Config struct {
    // ... basic config
    HTTP httpx.Config `yaml:"http"`
}
```

```yaml
# config.yaml
http:
  port: 8080
  security:
    cors:
      allowed_origins:
        - "http://localhost:3000"
      allowed_methods:
        - GET
        - POST
        - OPTIONS
```

### 4.2 Add Provider

```go
// pkg/app/provider.go

var SetupHTTPServing = []any{
    SetupHTTPServeMux,
    SetupHTTPListener,
    SetupProttpHandler,
    SetupHTTPServer,
    // SetupGRPCConn, // Add when using gRPC, for health check
}

// SetupGRPCConn - Add when using gRPC
// func SetupGRPCConn(lc *lifecycle.Lifecycle, grpcListener grpcx.Listener) (*grpc.ClientConn, error) {
//     return grpcx.SetupConnFactory("grpc-conn")(lc, &grpcx.ConnConfig{
//         Address:             grpcListener.Addr().String(),
//         LoadBalancingPolicy: grpcx.LoadBalancingPolicyPickFirst,
//     })
// }

func SetupHTTPServeMux() *http.ServeMux {
    return http.NewServeMux()
}

func SetupHTTPListener(lc *lifecycle.Lifecycle, conf *Config) (httpx.Listener, error) {
    return httpx.SetupListener(lc, &conf.HTTP)
}

func SetupHTTPServer(
    ctx context.Context,
    lc *lifecycle.Lifecycle,
    listener httpx.Listener,
    conf *Config,
    // conn *grpc.ClientConn, // Add when using gRPC
    mux *http.ServeMux,
    prottpHandler *prottpx.Handler,
) (http.Handler, *http.Server, error) {
    mux.Handle("/", prottpHandler)

    conf.HTTP.Security.CORS.AllowedHeaders = lo.Uniq(
        slices.Concat(
            conf.HTTP.Security.CORS.AllowedHeaders,
            i18nx.AllowHeaders,
            statusx.AllowHeaders,
            // auth.AllowHeaders,      // Add in Step 5
            // challenge.AllowHeaders, // Add in Step 5
        ),
    )

    handler := hook.Chain(
        server.DefaultMiddleware(kitlog.Default()),
        healthz.HTTPMiddleware(), // Without gRPC
        // healthz.HTTPMiddleware(healthz.WithGRPCHealthChecker(conn)), // Replace above line when using gRPC
        httpx.NoStore,
        httpx.Security(conf.HTTP.Security),
        // auth.Cookieize(conf.Auth.Cookie), // Add in Step 5
        normalize.HTTPMiddleware,
    )(mux)

    srv, err := httpx.SetupServerFactory("http-server", handler)(ctx, lc, &conf.HTTP, listener)
    if err != nil {
        return nil, nil, err
    }
    return handler, srv, nil
}

func SetupProttpHandler(i18n *i18nx.I18N, notifier errornotifier.Notifier) *prottpx.Handler {
    return prottpx.NewHandler(
        prottpx.ChainUnaryInterceptor(
            grpcServerInterceptorFactory(i18n, notifier),
            // auth.UnaryServerInterceptor(...), // Add in Step 5
        ),
    )
}
```

**Modifications when using gRPC:**

1. Add `SetupGRPCConn` to `SetupHTTPServing`
2. Uncomment the `SetupGRPCConn` function
3. Add `conn *grpc.ClientConn` parameter to `SetupHTTPServer`
4. Replace `healthz.HTTPMiddleware()` with `healthz.HTTPMiddleware(healthz.WithGRPCHealthChecker(conn))`

### 4.3 Register Business Service

```go
// pkg/app/provider.go

func SetupMyCustomerService(service *MyService, prottpHandler *prottpx.Handler) *MyCustomerService {
    customerService := NewCustomerService(service)
    myv1.RegisterMyCustomerServiceServer(prottpHandler, customerService)
    return customerService
}
```

### 4.4 Enable HTTP

```go
// cmd/myservice/main.go
providers := []any{
    // ...
    app.SetupHTTPServing,
    app.SetupMyCustomerService,
}
```

### 4.5 Checkpoint

After completing Step 4, verify the following items:

| #   | Check Item                                                               | Required | Verification Method                                                                      |
| --- | ------------------------------------------------------------------------ | -------- | ---------------------------------------------------------------------------------------- |
| 1   | Use `httpx.Security(conf.HTTP.Security)` to read config correctly        | ✅       | Confirm `hook.Chain` uses `httpx.Security(conf.HTTP.Security)`, not hardcoded config     |
| 2   | `conf.HTTP.Security.CORS.AllowedHeaders` includes `i18nx.AllowHeaders`   | ✅       | Confirm `slices.Concat(conf.HTTP.Security.CORS.AllowedHeaders, i18nx.AllowHeaders, ...)` |
| 3   | `conf.HTTP.Security.CORS.AllowedHeaders` includes `statusx.AllowHeaders` | ✅       | Confirm `slices.Concat(..., statusx.AllowHeaders)`                                       |
| 4   | Use `server.DefaultMiddleware` for request ID, logging, panic recovery   | ✅       | Confirm `hook.Chain` includes `server.DefaultMiddleware(kitlog.Default())`               |
| 5   | Use `healthz.HTTPMiddleware` for health check endpoint                   | ✅       | Confirm `hook.Chain` includes `healthz.HTTPMiddleware()`                                 |
| 6   | Use `httpx.NoStore` to set cache control header                          | ✅       | Confirm `hook.Chain` includes `httpx.NoStore`                                            |
| 7   | Use `normalize.HTTPMiddleware` for data normalization                    | ✅       | Confirm `hook.Chain` includes `normalize.HTTPMiddleware`                                 |
| 8   | `SetupProttpHandler` uses `grpcServerInterceptorFactory`                 | ✅       | Confirm `prottpx.NewHandler` calls `grpcServerInterceptorFactory(i18n, notifier)`        |

**Additional checks when using gRPC:**

| #   | Check Item                                            | Verification Method                                                           |
| --- | ----------------------------------------------------- | ----------------------------------------------------------------------------- |
| 9   | `SetupHTTPServing` includes `SetupGRPCConn`           | Confirm provider list contains `SetupGRPCConn`                                |
| 10  | `SetupHTTPServer` has `conn *grpc.ClientConn` param   | Confirm function signature includes this parameter                            |
| 11  | `healthz.HTTPMiddleware` uses `WithGRPCHealthChecker` | Confirm call is `healthz.HTTPMiddleware(healthz.WithGRPCHealthChecker(conn))` |

**If check fails:** Refer to section 4.2 code to fix `SetupHTTPServer` and `SetupProttpHandler`.

---

## Step 5: Integrate Auth

> Skip this step if user authentication is not needed.

### 5.1 Add Configuration

```go
// pkg/app/config.go
import "github.com/theplant/ciam-next/pkg/auth"

type Config struct {
    // ... other config
    Auth AuthConfig       `yaml:"auth"`
    CIAM grpcx.ConnConfig `yaml:"ciam"`
}

type AuthConfig struct {
    JWK    string            `yaml:"jwk"`
    Cookie auth.CookieConfig `yaml:"cookie"`
}
```

```yaml
# config.yaml
auth:
  jwk: "https://your-ciam-service/.well-known/jwks.json"
  cookie:
    name: "auth_token"
    # ... other cookie config

ciam:
  address: "ciam-service:9090"
```

### 5.2 Add Auth Provider

```go
// pkg/app/provider.go
import (
    "github.com/theplant/ciam-next/pkg/auth"
    "github.com/theplant/ciam-next/pkg/challenge"
    authv1 "github.com/theplant/ciam-next/gen/ciam/auth/v1"
)

type CIAMConn grpc.ClientConnInterface

var SetupAuthProviders = []any{
    SetupCIAMConn,
    SetupAuthServiceClient,
    SetupJWT,
}

func SetupCIAMConn(lc *lifecycle.Lifecycle, conf *Config) (CIAMConn, error) {
    return grpcx.SetupConnFactory("ciam-conn")(lc, &conf.CIAM)
}

func SetupAuthServiceClient(ciamConn CIAMConn) authv1.AuthServiceClient {
    return authv1.NewAuthServiceClient(ciamConn)
}

func SetupJWT(conf *Config) (*auth.JWT, error) {
    return auth.NewJWT[*auth.DefaultClaims](conf.Auth.JWK)
}
```

### 5.3 Modify HTTP Provider (Add Auth)

Modify `SetupProttpHandler` and `SetupHTTPServer` from Step 4:

```go
// pkg/app/provider.go

// SetupProttpHandler - Version with Auth
func SetupProttpHandler(
    i18n *i18nx.I18N,
    notifier errornotifier.Notifier,
    jwt *auth.JWT,                      // Added
    authClient authv1.AuthServiceClient, // Added
) *prottpx.Handler {
    return prottpx.NewHandler(
        prottpx.ChainUnaryInterceptor(
            grpcServerInterceptorFactory(i18n, notifier),
            // Added: Auth interceptor
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

// SetupHTTPServer - Version with Auth
func SetupHTTPServer(
    ctx context.Context,
    lc *lifecycle.Lifecycle,
    listener httpx.Listener,
    conf *Config,
    conn *grpc.ClientConn,
    mux *http.ServeMux,
    prottpHandler *prottpx.Handler,
) (http.Handler, *http.Server, error) {
    mux.Handle("/", prottpHandler)

    // Added: auth related headers
    conf.HTTP.Security.CORS.AllowedHeaders = lo.Uniq(
        slices.Concat(
            conf.HTTP.Security.CORS.AllowedHeaders,
            i18nx.AllowHeaders,
            statusx.AllowHeaders,
            auth.AllowHeaders,      // Added
            challenge.AllowHeaders, // Added
        ),
    )

    handler := hook.Chain(
        server.DefaultMiddleware(kitlog.Default()),
        healthz.HTTPMiddleware(healthz.WithGRPCHealthChecker(conn)),
        httpx.NoStore,
        httpx.Security(conf.HTTP.Security),
        auth.Cookieize(conf.Auth.Cookie), // Added
        normalize.HTTPMiddleware,
    )(mux)

    srv, err := httpx.SetupServerFactory("http-server", handler)(ctx, lc, &conf.HTTP, listener)
    if err != nil {
        return nil, nil, err
    }
    return handler, srv, nil
}
```

### 5.4 Enable Auth

```go
// cmd/myservice/main.go
providers := []any{
    // ...
    app.SetupAuthProviders, // Add this line
    app.SetupHTTPServing,
    // ...
}
```

### 5.5 Checkpoint

After completing Step 5, verify the following items:

| #   | Check Item                                              | Required | Verification Method                                                                                                       |
| --- | ------------------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------- |
| 1   | Config contains `AuthConfig` and `CIAM` config          | ✅       | Confirm `Config` struct has `Auth AuthConfig` and `CIAM grpcx.ConnConfig` fields                                          |
| 2   | Define `CIAMConn` type alias                            | ✅       | Confirm `type CIAMConn grpc.ClientConnInterface` exists                                                                   |
| 3   | `SetupAuthProviders` contains three providers           | ✅       | Confirm it includes `SetupCIAMConn`, `SetupAuthServiceClient`, `SetupJWT`                                                 |
| 4   | `SetupProttpHandler` has `jwt` and `authClient` params  | ✅       | Confirm function signature includes `jwt *auth.JWT` and `authClient authv1.AuthServiceClient`                             |
| 5   | `SetupProttpHandler` uses `auth.UnaryServerInterceptor` | ✅       | Confirm `prottpx.ChainUnaryInterceptor` includes `auth.UnaryServerInterceptor(...)`                                       |
| 6   | `SetupHTTPServer` uses `auth.Cookieize` middleware      | ✅       | Confirm `hook.Chain` includes `auth.Cookieize(conf.Auth.Cookie)`                                                          |
| 7   | CORS AllowedHeaders includes `auth.AllowHeaders`        | ✅       | Confirm `slices.Concat` includes `auth.AllowHeaders`                                                                      |
| 8   | CORS AllowedHeaders includes `challenge.AllowHeaders`   | ✅       | Confirm `slices.Concat` includes `challenge.AllowHeaders`                                                                 |
| 9   | Auth related packages imported                          | ✅       | Confirm imports include `github.com/theplant/ciam-next/pkg/auth`, `github.com/theplant/ciam-next/pkg/challenge`, `authv1` |

**If check fails:** Refer to sections 5.2 and 5.3 code to fix Auth Provider and HTTP Provider.

---

## Step 6: Output - Middleware Reference

After completing the integration, here are the middleware used at each layer:

### gRPC Interceptors

| Interceptor                                | Description                     | Dependency               |
| ------------------------------------------ | ------------------------------- | ------------------------ |
| `healthz.UnaryServerInterceptor`           | Health check for K8s probes     | `health.Server`          |
| `grpcx.DefaultErrorUnaryServerInterceptor` | Error reporting (Sentry)        | `errornotifier.Notifier` |
| `grpcx.RecoveryUnaryServerInterceptor`     | Panic recovery                  | None                     |
| `normalize.GRPCUnaryServerInterceptor`     | Request data normalization      | None                     |
| `logtracing.UnaryServerInterceptor`        | Logging and distributed tracing | None                     |
| `statusx.UnaryServerInterceptor`           | Error status i18n               | `i18nx.I18N`             |

**Interceptor execution order (request):**

```
healthz → errInter(outer) → normalize → logtracing → statusx → errInter(inner) → handler
```

### HTTP Middleware

| Middleware                 | Description                         | Condition |
| -------------------------- | ----------------------------------- | --------- |
| `server.DefaultMiddleware` | Request ID, logging, panic recovery | Required  |
| `healthz.HTTPMiddleware`   | Health check `/healthz`             | Required  |
| `httpx.NoStore`            | `Cache-Control: no-store`           | Required  |
| `httpx.Security`           | CORS and security headers           | Required  |
| `auth.Cookieize`           | Cookie to Authorization             | Auth only |
| `normalize.HTTPMiddleware` | Request data normalization          | Optional  |

**Middleware execution order (request):**

```
DefaultMiddleware → healthz → NoStore → Security → [Cookieize] → normalize → handler
```

### Prottp Handler Interceptors

| Interceptor                    | Description                      | Condition |
| ------------------------------ | -------------------------------- | --------- |
| `grpcServerInterceptorFactory` | Reuse gRPC interceptor chain     | Required  |
| `auth.UnaryServerInterceptor`  | JWT validation and session fetch | Auth only |

### CORS AllowedHeaders

| Headers                  | Source                   | Condition |
| ------------------------ | ------------------------ | --------- |
| `i18nx.AllowHeaders`     | Accept-Language, etc.    | Required  |
| `statusx.AllowHeaders`   | Error detail headers     | Required  |
| `auth.AllowHeaders`      | Authorization, etc.      | Auth only |
| `challenge.AllowHeaders` | MFA/verification related | Auth only |

---

## Complete Examples

### HTTP + gRPC + Auth

```go
// cmd/myservice/main.go
providers := []any{
    // Infrastructure
    app.SetupLogger,
    app.SetupErrorNotifier,
    app.SetupI18N,
    app.SetupDatabase,

    // Auth
    app.SetupAuthProviders,

    // Serving
    app.SetupGRPCServing,
    app.SetupHTTPServing,

    // Business services
    app.SetupMyService,
    app.SetupMyCustomerService,
}
```

### HTTP Only + Auth

```go
providers := []any{
    app.SetupLogger,
    app.SetupErrorNotifier,
    app.SetupI18N,

    app.SetupAuthProviders,
    app.SetupHTTPServing, // Use version without gRPC dependency

    app.SetupMyCustomerService,
}
```

### gRPC Only (No Auth)

```go
providers := []any{
    app.SetupLogger,
    app.SetupErrorNotifier,
    app.SetupI18N,

    app.SetupGRPCServing,

    app.SetupMyService,
}
```
