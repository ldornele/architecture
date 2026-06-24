---
Status: Active
Owner: HyperFleet Team
Last Updated: 2026-06-24
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

Baselines are captured from two environments. The Prow CI environment is the source of truth for thresholds enforced in CI. The GKE dev environment is useful for local development comparison.

## Environment

Both environments run the same stack: API, Sentinel (clusters + nodepools), 3 adapters (2 Kubernetes transport, 1 Maestro transport), and GCP Pub/Sub. All nodes are e2-standard-4 (4 vCPU / 16GB RAM) with a 5s Sentinel poll interval.

| Parameter | GKE dev | Prow CI (hyperfleet-dev-prow) |
|---|---|---|
| Execution | `kubectl run` (ClusterIP) | Prow `tier1-nightly` job |
| Database | ~1k seeded clusters | Fresh deploy per run |
| Date captured | 2026-06-18 | 2026-06-23 |


## Baselines

### API reads

API read thresholds are consistent across both environments.


| Operation                             | GKE dev | Prow CI | Threshold |
| ------------------------------------- | ------- | ------- | --------- |
| GET /clusters/{id} (small payload)    | 2.19ms  | 32.72ms | 50ms      |
| GET /clusters/{id} (medium payload)   | 2.82ms  | 30.99ms | 50ms      |
| GET /clusters/{id} (large payload)    | 4.34ms  | 32.61ms | 50ms      |
| GET /clusters (no filter)             | 5.10ms  | 32.63ms | 50ms      |
| GET /clusters (search filter)         | 4.89ms  | 33.03ms | 50ms      |
| GET /clusters (pageSize=10)           | 4.22ms  | 31.42ms | 50ms      |
| GET /clusters (page=1, pageSize=10)   | 4.23ms  | 33.30ms | 50ms      |


API read latencies on Prow are higher than GKE dev (~30ms vs ~3-5ms) but pass comfortably within the 50ms threshold.

### Reconciliation operations

Thresholds are calibrated from Prow CI baselines with ~50% headroom to absorb run-to-run variance on the shared cluster.


| Operation                               | GKE dev | Prow CI | Threshold |
| --------------------------------------- | ------- | ------- | --------- |
| Cluster create-to-reconciled            | 10.02s  | ~60s    | 90s       |
| Cluster update-to-re-reconciled         | 20.06s  | ~40s    | 60s       |
| Cluster delete-to-hard-delete           | 40.09s  | ~40s    | 60s       |
| Cluster cascade delete (with nodepools) | 40.08s  | ~50s    | 75s       |
| NodePool create-to-reconciled           | 10.02s  | ~20s    | 30s       |
| NodePool delete-to-hard-delete          | 20.05s  | ~20s    | 30s       |


Prow CI latencies are higher than GKE dev due to shared cluster resources.

## How to reproduce

### Prow CI

Prow baselines are captured automatically by the `tier1-nightly` job. To trigger a run on demand, use the [Gangway API](https://github.com/openshift-hyperfleet/architecture/blob/main/hyperfleet/docs/release/test-release/trigger-e2e-jobs-via-gangway.md). Results are available in the [Prow dashboard](https://prow.ci.openshift.org/?job=periodic-ci-openshift-hyperfleet-hyperfleet-e2e-main-e2e-tier1-nightly) build logs (search for `[PERF]` lines).

### GKE dev

See the [perf README](https://github.com/openshift-hyperfleet/hyperfleet-e2e/blob/main/perf/README.md) in the `hyperfleet-e2e` repo for prerequisites, infrastructure setup, seeding, and test execution instructions.
