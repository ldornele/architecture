---
Status: Active
Owner: HyperFleet Team
Last Updated: 2026-05-11
---

# Release Documentation

## Start Here

| Document | Purpose |
|----------|---------|
| [Konflux Release Pipeline Design](./konflux-release-pipeline-design.md) | How the build and release pipeline works — architecture, flows, and design decisions |
| [HyperFleet Release Process](./hyperfleet-release-process.md) | The release process — cadence, checklists, branching, bug handling, hotfixes |
| [Helm OCI Distribution Design](./helm-oci-distribution-design.md) | How Helm charts are published as OCI artifacts via Konflux |
| [Glossary](./glossary.md) | Definitions of terms used across the release docs |
| [ADR 0014](../../adrs/0014-konflux-build-and-release.md) | Decision record for adopting Konflux |
| [ADR 0016](../../adrs/0016-helm-oci-distribution.md) | Decision record for Helm OCI distribution |

## Prow Test and Release

The `test-release/` subdirectory contains Prow-specific docs for CI job setup and E2E testing infrastructure.
