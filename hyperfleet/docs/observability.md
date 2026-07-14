---
Status: Active
Owner: HyperFleet Platform Team
Last Updated: 2026-07-13
---

# HyperFleet Observability Reference

Consolidated reference for metrics and monitoring across all HyperFleet components.

---

## Table of Contents

1. [Overview](#overview)
2. [Component Metrics Endpoints](#component-metrics-endpoints)
3. [Component-Specific Metrics](#component-specific-metrics)
4. [References](#references)

---

## Overview

HyperFleet components expose Prometheus-compatible metrics for monitoring and observability. All metrics follow the [HyperFleet Metrics Standard](../standards/metrics.md) for consistent naming and labeling.

**Key Points:**

- All components expose metrics on **port 9090** at `/metrics` endpoint
- Metrics follow the `hyperfleet_<component>_<metric_name>_<unit>` naming convention
- All metrics include standard labels: `component` and `version`
- Individual component repositories contain detailed metric definitions

---

## Component Metrics Endpoints

All HyperFleet components expose metrics on **port 9090** at the `/metrics` endpoint.

| Component | Repository | Metrics Endpoint | Health Endpoints |
|-----------|------------|------------------|------------------|
| **API** | [hyperfleet-api](https://github.com/openshift-hyperfleet/hyperfleet-api) | `:9090/metrics` | `:8080/healthz`, `:8080/readyz` |
| **Sentinel** | [hyperfleet-sentinel](https://github.com/openshift-hyperfleet/hyperfleet-sentinel) | `:9090/metrics` | `:8080/healthz`, `:8080/readyz` |
| **Adapter Framework** | [hyperfleet-adapter](https://github.com/openshift-hyperfleet/hyperfleet-adapter) | `:9090/metrics` | `:8080/healthz`, `:8080/readyz` |

**Note:** Metrics port (`:9090`) is separate from health port (`:8080`) for network isolation. The API service also exposes REST endpoints on port `:8000`.

---

## Component-Specific Metrics

Detailed metric definitions are documented in each component's repository:

### API Service

- **Repository**: [hyperfleet-api](https://github.com/openshift-hyperfleet/hyperfleet-api)
- **Metrics Documentation**: `docs/metrics.md` in repository
- **Architecture Doc**: [api-service.md](../components/api-service/api-service.md)
- **Metrics Port**: `:9090/metrics`

### Sentinel

- **Repository**: [hyperfleet-sentinel](https://github.com/openshift-hyperfleet/hyperfleet-sentinel)
- **Metrics Documentation**: `docs/metrics.md` in repository
- **Architecture Doc**: [sentinel.md](../components/sentinel/sentinel.md)
- **Metrics Port**: `:9090/metrics`

### Adapter Framework

- **Repository**: [hyperfleet-adapter](https://github.com/openshift-hyperfleet/hyperfleet-adapter)
- **Metrics Documentation**: [adapter-metrics.md](../components/adapter/framework/adapter-metrics.md) (architecture repo copy; also available in component repository)
- **Architecture Doc**: [adapter-deployment.md](../components/adapter/framework/adapter-deployment.md)
- **Metrics Port**: `:9090/metrics`

---

## References

### HyperFleet Standards

- [Metrics Standard](../standards/metrics.md) — Naming conventions, required labels, best practices
- [Health Endpoints Standard](../standards/health-endpoints.md) — Health check conventions
- [Logging Specification](../standards/logging-specification.md) — Structured logging format
- [Tracing Standard](../standards/tracing.md) — Distributed tracing conventions

### Component Repositories

For detailed metric definitions, alert rules, and monitoring setup instructions:

- [hyperfleet-api](https://github.com/openshift-hyperfleet/hyperfleet-api) — API service metrics and alerts
- [hyperfleet-sentinel](https://github.com/openshift-hyperfleet/hyperfleet-sentinel) — Sentinel metrics and alerts
- [hyperfleet-adapter](https://github.com/openshift-hyperfleet/hyperfleet-adapter) — Adapter framework metrics and alerts
