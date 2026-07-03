---
Status: Active
Owner: HyperFleet Team
Last Updated: 2026-06-26
---

# Performance Baselines

**Jira**: [HYPERFLEET-1185](https://issues.redhat.com/browse/HYPERFLEET-1185), [HYPERFLEET-1270](https://issues.redhat.com/browse/HYPERFLEET-1270)

---

## Table of Contents

- [Overview](#overview)
- [Environment](#environment)
- [Baselines](#baselines)
  - [API reads](#api-reads)
  - [Reconciliation operations](#reconciliation-operations)
- [How to reproduce](#how-to-reproduce)

## Overview

Lightweight performance baselines for core HyperFleet operations, captured for the v1.0.0 release. These baselines measure the overhead HyperFleet introduces, not infrastructure capacity.

Baselines are captured from two environments. The Prow CI environment provides the baselines from which CI thresholds are derived. The GKE dev environment is useful for local development comparison.

## Environment

Both environments run the same stack: API, Sentinel (clusters + nodepools), 3 adapters (2 Kubernetes transport, 1 Maestro transport), and GCP Pub/Sub. All nodes are e2-standard-4 (4 vCPU / 16GB RAM) with a 5s Sentinel poll interval.

| Parameter | GKE dev | Prow CI (hyperfleet-dev-prow) |
|---|---|---|
| Execution | `kubectl run` (ClusterIP) | Prow `tier1-nightly` job |
| Database | ~1k seeded clusters | Fresh deploy per run |
| Date captured | 2026-06-18 | 2026-06-23 |

## Baselines

### API reads

| Operation                             | GKE dev | Prow CI |
| ------------------------------------- | ------- | ------- |
| GET /clusters/{id} (small payload)    | 2.19ms  | 32.72ms |
| GET /clusters/{id} (medium payload)   | 2.82ms  | 30.99ms |
| GET /clusters/{id} (large payload)    | 4.34ms  | 32.61ms |
| GET /clusters (no filter)             | 5.10ms  | 32.63ms |
| GET /clusters (search filter)         | 4.89ms  | 33.03ms |
| GET /clusters (size=10)               | 4.22ms  | 31.42ms |
| GET /clusters (page=1, size=10)       | 4.23ms  | 33.30ms |

### Reconciliation operations

| Operation                               | GKE dev | Prow CI |
| --------------------------------------- | ------- | ------- |
| Cluster create-to-reconciled            | 10.02s  | ~60s    |
| Cluster update-to-re-reconciled         | 20.06s  | ~40s    |
| Cluster delete-to-hard-delete           | 40.09s  | ~40s    |
| Cluster cascade delete (with nodepools) | 40.08s  | ~50s    |
| NodePool create-to-reconciled           | 10.02s  | ~20s    |
| NodePool delete-to-hard-delete          | 20.05s  | ~20s    |

Prow CI latencies are higher than GKE dev due to shared cluster resources.

CI thresholds for both API reads and reconciliation operations are defined in [`pkg/config/thresholds.go`](https://github.com/openshift-hyperfleet/hyperfleet-e2e/blob/main/pkg/config/thresholds.go) in the `hyperfleet-e2e` repo. Thresholds apply a margin over Prow CI baselines to absorb run-to-run variance.

## How to reproduce

### Prow CI

Prow baselines are captured automatically by the `tier1-nightly` job. To trigger a run on demand, use the [Gangway API](https://github.com/openshift-hyperfleet/architecture/blob/main/hyperfleet/docs/release/test-release/trigger-e2e-jobs-via-gangway.md). Results are available in the [Prow dashboard](https://prow.ci.openshift.org/?job=periodic-ci-openshift-hyperfleet-hyperfleet-e2e-main-e2e-tier1-nightly) build logs (search for `[PERF]` lines).

### GKE dev

See the [perf README](https://github.com/openshift-hyperfleet/hyperfleet-e2e/blob/main/perf/README.md) in the `hyperfleet-e2e` repo for prerequisites, infrastructure setup, seeding, and test execution instructions.
