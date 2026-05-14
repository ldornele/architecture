---
Status: Active
Owner: Ciaran Roche
Last Updated: 2026-04-21
---

# 0014 — Konflux for Container Image Build and Release

## Context

HyperFleet builds container images for three components (API, Sentinel, Adapter) using Prow's ci-operator, publishing to `registry.ci.openshift.org`. This setup lacks supply chain security: no signed provenance records, no SBOM generation, no image signing.

Red Hat requires all services to use Konflux for image builds to achieve SLSA Level 3 compliance. Konflux provides provenance via Tekton Chains, automatic SBOM generation, image signing with cosign, and Enterprise Contract policy enforcement — capabilities Prow does not offer.

Meanwhile, HyperFleet's E2E testing infrastructure runs on Prow with GKE clusters, GCP credentials, and an established test framework that would be costly and complex to replicate in Konflux.

## Decision

Adopt a hybrid model: **Konflux handles image build and release, Prow retains E2E testing.**

The architecture has three layers:

- **PaC (Pipelines as Code)** — trigger layer. Watches GitHub webhooks and matches git events (push to main, version tags) to `.tekton/` pipeline files in each component repo.
- **Konflux** — orchestration layer. Builds images, generates SBOM, signs provenance (Tekton Chains), runs Enterprise Contract validation, and releases to `quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/`.
- **Prow** — test layer. Runs nightly and RC E2E tests against Konflux-built images from Quay. Retains PR presubmit checks (lint, unit tests).

Key design choices:

- One Application (`hyperfleet`) with three Components, releasing through a single ReleasePlanAdmission with auto-release (`block-releases: false`)
- Tag-based triggers using CEL expressions (not globs) to distinguish RC tags from release tags
- `app-interface-standard` Enterprise Contract policy for service-type releases
- RC E2E triggered via `hyperfleet-release` repo tag -> GitHub Action -> Gangway -> Prow
- `rh-push-to-external-registry` release pipeline with Pyxis registration and Slack notifications

See [Konflux Release Pipeline Design](../docs/release/konflux-release-pipeline-design.md) for the full design document.

## Consequences

**Gains:**

- SLSA Level 3 supply chain security (provenance, SBOM, image signing) with zero extra configuration
- Simplified RC testing — one reusable Prow job with parameterized image coordinates replaces per-release-branch pipeline definitions
- Auto-release on merge to main eliminates manual image publishing
- Public Quay images remove pull secret complexity for Prow and partner environments
- Alignment with Red Hat release engineering standards

**Trade-offs:**

- Two build systems during transition (Konflux for images, Prow for PR validation and E2E)
- RC E2E triggering depends on Gangway API availability
- Team must learn Konflux primitives (RPA, ITS, PaC, Snapshots)

## Alternatives Considered

| Alternative | Why Rejected |
|-------------|--------------|
| Full Prow (status quo) | No path to SLSA compliance. No provenance, SBOM, or signing. Does not meet Red Hat release engineering requirements. |
| Full Konflux (including E2E) | Would require replicating GKE cluster provisioning, GCP credentials, and the entire hyperfleet-e2e framework inside Konflux ITS pipelines. High cost, low benefit — Prow already works. |
| Hybrid with ITS wrapper for E2E | An IntegrationTestScenario that triggers Prow via Gangway and polls for results. Adds complexity (polling, timeouts, error handling) as a Tekton pipeline when a simpler GitHub Action achieves the same result with better DX (retrigger via `gh workflow run`). Deferred to fast-follow if automated gating is needed. |
| Separate RPAs per build context | One RPA for nightly (auto-release) + one for releases (gated via `block-releases`). Rejected because `block-releases` is per-RPA not per-Snapshot — every build from the Application would create blocked releases on the gated RPA, generating noise without meaningful safety. |
| Glob patterns for tag matching | PaC glob patterns (e.g., `v*`) cannot distinguish `v1.0.0-rc1` from `v1.0.0`. CEL expressions with regex anchoring are required. Verified against PaC source code and hypershift production configuration. |
