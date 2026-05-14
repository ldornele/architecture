---
Status: Active
Owner: HyperFleet Architecture Team
Last Updated: 2026-05-14
---

# 0003 — OpenAPI 3.0 as the System Contract (API-First Development)

## Context

Sentinel and every Adapter consume the HyperFleet REST API. External tooling and future third-party integrations will also depend on it. Without a machine-readable contract reviewed before implementation, API shape tends to drift: field names change across handlers, error responses are inconsistent, and consumers must reverse-engineer the API from source code. HYPERFLEET-18 established that "getting the OpenAPI specification correct from the start is paramount."

## Decision

The **OpenAPI 3.0.3 specification** authored in [`hyperfleet-api-spec`](https://github.com/openshift-hyperfleet/hyperfleet-api-spec) (TypeSpec) is the single source of truth for the HyperFleet API contract. Code is generated from the spec using **`oapi-codegen`**. Key conventions:

- The spec is authored in **TypeSpec** in `hyperfleet-api-spec` and compiled to OpenAPI 3.0.3; published as a Go module (`github.com/openshift-hyperfleet/hyperfleet-api-spec`).
- Both `hyperfleet-api` and `hyperfleet-sentinel` consume the spec via the Go module — `openapi/openapi.yaml` is extracted from the module cache at code-generation time and is **not tracked in git** in either repository.
- Generated Go code (model structs, HTTP client, embedded spec) is **not committed to git**; developers run `make generate` (or `make generate-all`) after clone.
- In `hyperfleet-api`, the compiled spec is **embedded in the binary** (`//go:embed`) and served at `/api/hyperfleet/v1/openapi` at runtime.
- All API errors conform to **RFC 9457 Problem Details** (`application/problem+json`) with structured `HYPERFLEET-CAT-NUM` error codes.
- The API is versioned under the `/api/hyperfleet/v1/` prefix; no version negotiation is needed during the MVP lifecycle.

For the detailed mechanics of schema import and code generation, see [openapi/README.md](https://github.com/openshift-hyperfleet/hyperfleet-api/blob/main/openapi/README.md) in `hyperfleet-api` and [openapi/README.md](https://github.com/openshift-hyperfleet/hyperfleet-sentinel/blob/main/openapi/README.md) in `hyperfleet-sentinel`.

## Consequences

**Gains:** All consumers (Sentinel, Adapters, CLI tools) generate type-safe clients directly from the spec; the embedded spec enables contract testing against the live binary; RFC 9457 error responses are consistent across all endpoints; spec reviews surface API design issues before any handler is written; schema is consumed from a versioned Go module — no manual file copying required.

**Trade-offs:** Generated code is not in git, so a broken codegen tool blocks all development; the TypeSpec → OpenAPI compilation step adds a tooling dependency; developers unfamiliar with `oapi-codegen` must learn its opinionated code layout before contributing.

## Alternatives Considered

| Alternative | Why Rejected |
|-------------|--------------|
| gRPC / Protobuf | REST with JSON is better suited for HTTP-native clients and third-party tooling; the team's operator audience is more familiar with REST; Kubernetes-style API conventions (conditions, labels, pagination) map naturally to REST |
| Hand-crafted Go types (no spec) | Drift between documented API and implementation is inevitable; no auto-generated clients for Sentinel/Adapter; every schema change requires updating multiple files manually |
| openapi-generator-cli | `oapi-codegen` produces leaner, idiomatic Go with fewer dependencies; `openapi-generator-cli` requires a JVM runtime in the developer environment |
| GraphQL | Adds a query language layer that is not needed for the CRUD-heavy cluster lifecycle domain; fewer off-the-shelf Red Hat tooling integrations |
| Committing `openapi.yaml` to `hyperfleet-api` | Creates a second copy that can drift from `hyperfleet-api-spec`; module-based consumption guarantees the working spec always matches the declared version in `go.mod` |
