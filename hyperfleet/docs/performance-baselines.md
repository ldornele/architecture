---
Status: Active
Owner: HyperFleet Team
Last Updated: 2026-06-18
---

# Performance Baselines

**Jira**: [HYPERFLEET-1185](https://issues.redhat.com/browse/HYPERFLEET-1185)

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

## Environment

| Parameter | Value |
|---|---|
| Cluster | GKE dev (e2-standard-4 nodes, 4 vCPU / 16GB RAM) |
| Execution | In-cluster pod via `kubectl run` (ClusterIP, no external network) |
| Database | PostgreSQL with ~1k seeded clusters |
| Sentinel poll interval | 5s |
| Components | API, Sentinel (clusters + nodepools), 3 adapters, GCP Pub/Sub |
| Date captured | 2026-06-18 |

## Baselines

### API reads

| Operation | Measured | Threshold |
|---|---|---|
| GET /clusters/{id} (small payload) | 2.19ms | 50ms |
| GET /clusters/{id} (medium payload) | 2.82ms | 50ms |
| GET /clusters/{id} (large payload) | 4.34ms | 50ms |
| GET /clusters (no filter) | 5.10ms | 50ms |
| GET /clusters (search filter) | 4.89ms | 50ms |
| GET /clusters (pageSize=10) | 4.22ms | 50ms |
| GET /clusters (page=1, pageSize=10) | 4.23ms | 50ms |

### Reconciliation operations

| Operation | Measured | Threshold |
|---|---|---|
| Cluster create-to-reconciled | 10.02s | 20s |
| Cluster update-to-re-reconciled | 20.06s | 30s |
| Cluster delete-to-hard-delete | 40.09s | 60s |
| Cluster cascade delete (with nodepools) | 40.08s | 60s |
| NodePool create-to-reconciled | 10.02s | 20s |
| NodePool delete-to-hard-delete | 20.05s | 30s |

Thresholds include buffer for variance across runs and environments.

## How to reproduce

See the [perf README](https://github.com/openshift-hyperfleet/hyperfleet-e2e/blob/main/perf/README.md) in the `hyperfleet-e2e` repo for prerequisites, infrastructure setup, seeding, and test execution instructions.
