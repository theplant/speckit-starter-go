---
description: Add IAM integration workflow (OpenAPI registration + spec annotations + tests) for a Go service
---

## User Input

```text
$ARGUMENTS
```

Use the user input (if provided) to determine:
- Target service name (e.g. console/admin)
- OpenAPI spec file path (relative to repo root)
- IAM private URL to use in local/dev
- Service code and permission namespace (e.g. `pim`, `loyalty`)

## Goal

Implement IAM integration in a Go service by:
1. Adding IAM config (`privateURL`, `openAPISpecPath`)
2. Registering the service OpenAPI spec to IAM during startup
3. Ensuring OpenAPI spec contains required IAM annotations (`x-iam-service`, `x-iam-permissions`, per-operation `x-iam-access`, `x-required-permission`)
4. Adding tests to prevent regressions

This workflow is based on the integration pattern used in `theplant/loyalty` commit `62c11e0a49e5d36e23f6fe956ff5757e07584164`.

## Required Inputs (decide before editing)

- **Service code**: a stable code string used by IAM, e.g. `pim`, `loyalty`
- **OpenAPI spec path**: relative to repo root, e.g. `pkg/console/0000-openapi-<service>-api.yaml`
- **Permission namespace**: usually the same as service code, e.g. `pim:*`

## Implementation Steps

### 1. Add IAM config fields to your service config

In the service `Config` struct (example naming):

- Add a top-level field:
  - `IAM IAMConfig \`confx:"iam" usage:"IAM integration configuration"\``

- Add a new struct:
  - `type IAMConfig struct {`
    - `PrivateURL string \`confx:"privateURL" usage:"IAM private API URL for OpenAPI registration"\``
    - `OpenAPISpecPath string \`confx:"openAPISpecPath" usage:"Path to OpenAPI spec file for IAM registration"\``
    - `}`

In the default embedded YAML config:

- Set `iam.privateURL` to empty by default (disabled)
- Set `iam.openAPISpecPath` to empty by default (use hardcoded default path)

### 2. Add IAM OpenAPI registration actor

Create a file in your service package (example: `pkg/<service>/app/iam.go`) that:

- Defines a constant default spec path (relative to repo root)
- Resolves the spec path:
  - If empty: use default
  - If absolute: keep
  - If relative: resolve against project root using `runtime.Caller(0)` and `filepath.Join(..., "../../..")`
- On startup (via lifecycle):
  - Read spec file into string
  - Create IAM registry client
  - Call `RegisterOpenAPI` to register spec
  - Log the result

Reference logic (adapt paths to your project):
- `theplant/loyalty/pkg/console/app/iam.go`

### 3. Wire IAM registration into startup setup

Ensure the registration function is called during app lifecycle setup.

Typical patterns:
- If you have a provider list (DI): append `app.SetupIAMRegistration`
- If you have a `Setup()` function: call `SetupOpenAPIRegistration(lc, conf)`

Reference wiring:
- `theplant/loyalty/cmd/console/setup/setup.go`
- `theplant/loyalty/pkg/console/app/provider.go` (`var SetupIAMRegistration = SetupOpenAPIRegistration`)

### 4. Add Go module dependencies

Add the IAM client dependencies to `go.mod`:

- `github.com/theplant/iam`
- `github.com/theplant/iam/api/private`
- `github.com/theplant/inject` (for `lifecycle`)

Run:

```sh
go mod tidy
```

### 5. Update OpenAPI spec with IAM annotations

In your OpenAPI YAML (example path: `pkg/<service>/0000-openapi-...yaml`):

- Add service metadata:
  - `x-iam-service:`
    - `code: <serviceCode>`

- Add permission definitions:
  - `x-iam-permissions:`
    - a list of permissions used by the service

- For every operation under `paths`:
  - Add `x-iam-access: authorized`
  - Add `x-required-permission: <permission>`

The tests below will enforce:
- `x-iam-service` exists
- expected service code exists
- `x-iam-permissions` exists
- every operation has `x-iam-access` and `x-required-permission`

### 6. Add tests to lock the integration

Create tests in the same package where config/default yaml is embedded (example: `pkg/<service>/app/iam_test.go`).

Tests to include:

1. **Config default test**:
   - Unmarshal embedded default config
   - Assert `IAM.PrivateURL == ""`
   - Assert `IAM.OpenAPISpecPath == ""`

2. **Spec annotation presence test** (string contains):
   - Read the OpenAPI YAML
   - Assert contains `x-iam-service:`
   - Assert contains `code: <serviceCode>`
   - Assert contains `x-iam-permissions:`
   - Assert contains at least the key permissions you expect (optional but recommended)

3. **All endpoints annotated test** (YAML parse):
   - `yaml.Unmarshal` into `map[string]any`
   - Iterate `paths` and each HTTP method
   - For each operation (with `operationID`):
     - assert `x-iam-access` exists and equals `authorized`
     - assert `x-required-permission` exists

4. **Spec path resolver test**:
   - Ensure `resolveSpecPath("")` returns an absolute path ending with your default spec path

Reference tests:
- `theplant/loyalty/pkg/console/app/iam_test.go`

### 7. Manual verification flow (local)

1. Start IAM locally (whatever your org uses) and obtain the private URL, e.g.
   - `http://localhost:4001`

2. Configure your service:

```yaml
iam:
  privateURL: http://localhost:4001
  openAPISpecPath: ""  # or explicit path
```

3. Run your service.

4. Verify logs show:
- registered OpenAPI with IAM
- includes `service_code`, `parsed_operations`, `upserted_endpoints`

5. Run tests:

```sh
go test ./...
```

## Notes / Common Failure Points

- **Spec path wrong**: `resolveSpecPath` relies on file layout. If your `iam.go` is not at `pkg/<service>/app/iam.go`, update the `../../..` traversal.
- **OpenAPI missing `x-required-permission`**: the YAML endpoint test will fail.
- **IAM is optional**: keep `privateURL` empty by default so dev/test doesn’t require IAM running.

## Completion Criteria

- Service starts without IAM configured
- With IAM configured (`privateURL`), service registers OpenAPI successfully
- `go test ./...` passes and enforces IAM annotations
