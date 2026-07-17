---
Status: Active
Owner: HyperFleet Platform Team
Last Updated: 2026-07-13
---

# HyperFleet Observability Reference

Consolidated reference for metrics and monitoring across HyperFleet's deployable components.

---

## Table of Contents

- [Overview](#overview)
- [Component Metrics and Health Endpoints](#component-metrics-and-health-endpoints)
- [References](#references)

---

## Overview

HyperFleet components expose Prometheus-compatible metrics for monitoring and observability. All metrics follow the [HyperFleet Metrics Standard](../standards/metrics.md) for consistent naming and labeling.

**Key Points:**

- All components expose metrics on **port 9090** at `/metrics` endpoint
- Metrics follow the `hyperfleet_<component>_<metric_name>_<unit>` naming convention
- All metrics include standard labels: `component` and `version`
- Individual component repositories contain detailed metric definitions

---

## Component Metrics and Health Endpoints

All HyperFleet components expose:

- **Metrics** on port **9090** at `/metrics`
- **Health status** on port **8080** at `/healthz` (liveness) and `/readyz` (readiness)

| Component | Repository | Metrics Endpoint | Health Endpoints | Metrics Docs | Architecture Doc |
|-----------|------------|------------------|------------------|--------------|------------------|
| **API** | [hyperfleet-api](https://github.com/openshift-hyperfleet/hyperfleet-api) | `:9090/metrics` | `:8080/healthz`, `:8080/readyz` | [docs/metrics.md](https://github.com/openshift-hyperfleet/hyperfleet-api/blob/main/docs/metrics.md) | [api-service.md](../components/api-service/api-service.md) |
| **Sentinel** | [hyperfleet-sentinel](https://github.com/openshift-hyperfleet/hyperfleet-sentinel) | `:9090/metrics` | `:8080/healthz`, `:8080/readyz` | [docs/metrics.md](https://github.com/openshift-hyperfleet/hyperfleet-sentinel/blob/main/docs/metrics.md) | [sentinel.md](../components/sentinel/sentinel.md) |
| **Adapter Framework** | [hyperfleet-adapter](https://github.com/openshift-hyperfleet/hyperfleet-adapter) | `:9090/metrics` | `:8080/healthz`, `:8080/readyz` | [docs/metrics.md](https://github.com/openshift-hyperfleet/hyperfleet-adapter/blob/main/docs/metrics.md) | [adapter-deployment.md](../components/adapter/framework/adapter-deployment.md) |

**Note:** Metrics port (`:9090`) is separate from health port (`:8080`) for network isolation. The API service also exposes REST endpoints on port `:8000`.

---

## References

- [Metrics Standard](../standards/metrics.md) — Naming conventions, required labels, best practices
- [Health Endpoints Standard](../standards/health-endpoints.md) — Health check conventions
- [Logging Specification](../standards/logging-specification.md) — Structured logging format
- [Tracing Standard](../standards/tracing.md) — Distributed tracing conventions
