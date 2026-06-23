# AGENTS.md

This file provides guidance to AI agents when working with code in this repository.

## What This Is

The OpenShift oauth-apiserver is a Kubernetes aggregated API server serving the `oauth.openshift.io` and `user.openshift.io` API groups. It also acts as a Kubernetes Webhook Token Authenticator. It runs as part of the OpenShift control plane, proxied by kube-apiserver, and shares etcd for storage.

The binary has two subcommands: `start` (runs the aggregated API server) and `external-oidc` (runs a standalone webhook token authenticator for external OIDC).

## Build and Test Commands

```sh
make build              # Build binaries (oauth-apiserver + oauth-apiserver-tests-ext)
make test               # Run unit tests (./pkg/... ./cmd/...)
make verify             # Verify generated files are up-to-date
make update             # Re-generate conversions, deep copies, defaulters, openapi
make test-e2e           # Run e2e tests (requires a running OpenShift cluster)
```

Run a single unit test:
```sh
go test ./pkg/tokenvalidation/... -run TestMyFunction -count=1
```

Run a specific e2e test:
```sh
WHAT=TestName make run-e2e-test
```

## Architecture

### Server Composition (Delegation Chain)

The server uses Kubernetes API server delegation chaining. In `pkg/apiserver/apiserver.go`, the `New()` method builds the chain bottom-up:

1. **Empty delegate** (base)
2. **OAuth API server** (`pkg/oauth/apiserver/`) — registers `oauth.openshift.io` resources
3. **User API server** (`pkg/user/apiserver/`) — registers `user.openshift.io` resources
4. **Top-level GenericAPIServer** — ties them together

Each sub-server gets a shallow-copied `GenericConfig` with PostStartHooks cleared to prevent double registration.

### API Group Structure

Each API group follows the same internal layout:

- `pkg/{oauth,user}/apis/{group}/` — internal (hub) types
- `pkg/{oauth,user}/apis/{group}/v1/` — versioned types and conversions
- `pkg/{oauth,user}/apis/{group}/install/` — scheme registration
- `pkg/{oauth,user}/apis/{group}/validation/` — validation logic
- `pkg/{oauth,user}/apiserver/` — API server wiring, installs REST storage
- `pkg/{oauth,user}/apiserver/registry/` — per-resource REST storage and strategy implementations (each resource has a `strategy.go` + `etcd/` subdirectory)

The `pkg/api/install/` package registers both groups into the server's scheme.

### External OIDC (`pkg/externaloidc/`)

A separate subsystem for external OIDC authentication, invoked via the `external-oidc` subcommand. It runs a standalone HTTPS webhook server (not the aggregated API server). Key sub-packages:

- `apis/authentication/` — internal types, versioned types (v1alpha1), validation, conversion for external OIDC config
- `authenticator/jwt/` — JWT authenticator with external claim sourcing
- `cel/` — CEL expression compilation for claim mappings and external source URLs
- `oidc/externalclaims/resolver/` — fetches claims from external HTTP sources at authentication time
- `server/` — standalone HTTPS server for webhook token authentication
- `cmd/` — cobra command wiring

### Token Validation (`pkg/tokenvalidation/`)

Handles OAuth access token authentication and timeout-based invalidation. The `TimeoutValidator` runs a background goroutine that periodically flushes expired tokens.

### Etcd Storage Prefixes

Etcd prefixes are hardcoded for backward compatibility with historical OpenShift data (e.g., `oauth/accesstokens`, `useridentities`). These cannot be changed without a data migration. See `specialDefaultResourcePrefixes` in `pkg/cmd/oauth-apiserver/cmd.go`.

## Code Conventions

- Follow [Kubernetes Code Conventions](https://github.com/kubernetes/community/blob/main/contributors/guide/coding-conventions.md).
- All exported and unexported types/functions/methods should have descriptive Go doc comments.
- Wrap errors with meaningful context. Follow [Kubernetes Logging Conventions](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-instrumentation/logging.md).
- Use table-driven tests. All changes must include unit tests.
- Use dashes (not underscores) in command-line flags.
- Vendor dependencies with `go mod vendor`. Do not add new dependencies without strong justification.

## Generated Files

After any API type changes (including `openshift/api` dependency bumps), run `make verify` to check and `make update` to regenerate. Generated files include conversions, deep copies, defaulters, and OpenAPI definitions. The generators are invoked via `hack/update-generated-*.sh` scripts.

## Key Reference Files

- `ARCHITECTURE.md` — detailed architecture, trade-offs, and design decisions with line-number references
- `CONTRIBUTING.md` — full code conventions, testing guidelines, PR process, and CI/CD details

## CI

OpenShift uses Prow for CI. Job configs live in [openshift/release](https://github.com/openshift/release/tree/main/ci-operator/config/openshift). PR titles should be prefixed with a Jira ticket (e.g., `CNTRLPLANE-XXXX:`) or `NO-JIRA:`. Use `/retest` to retry flaky checks.
