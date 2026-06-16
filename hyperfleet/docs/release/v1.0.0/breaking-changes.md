---
Status: Active
Owner: HyperFleet Team
Last Updated: 2026-06-10
---

# v0.2.0 to v1.0.0 Breaking-Change and Reconfiguration Checklist

> **Audience:** Internal partner teams (GCP, ROSA) upgrading from HyperFleet experimental (v0.2.0) to v1.0.0, and the HyperFleet release team.

## Overview

This document is the deliverable for [HYPERFLEET-1177](https://redhat.atlassian.net/browse/HYPERFLEET-1177). It lists every breaking change and required reconfiguration step between v0.2.0 and v1.0.0 so partner teams can reconfigure and redeploy with no surprises.

## Breaking Changes

### API Contract

| # | Change | Partner Action | Impact if Missed | Classification | Ticket | Parent |
|---|--------|---------------|-----------------|----------------|--------|--------|
| 1 | **Status report: POST to PUT** for clusters and nodepools | Change HTTP method in all adapter post-actions from POST to PUT | **405 Method Not Allowed.** PUT-only routes confirmed in `plugins/clusters/plugin.go`; no POST route exists | Automatable (1178) | [HYPERFLEET-978](https://redhat.atlassian.net/browse/HYPERFLEET-978) | - |
| 2 | **Ready condition removed; replaced by Reconciled** | Replace all `Ready` references with `Reconciled` in status queries, scripts, and monitoring | **Silent hang.** API silently accepts `type="Ready"` in status reports but never aggregates it into resource conditions; scripts polling for Ready=True wait forever | Automatable (1178) | [HYPERFLEET-1052](https://redhat.atlassian.net/browse/HYPERFLEET-1052) | [HYPERFLEET-559](https://redhat.atlassian.net/browse/HYPERFLEET-559) |
| 3 | **Aggregated condition Available renamed to LastKnownReconciled** | Replace `Available` with `LastKnownReconciled` in all resource-level condition queries | **Not found.** Aggregation code produces only `Reconciled` and `LastKnownReconciled`; resource-level `Available` is no longer emitted | Automatable (1178) | [HYPERFLEET-1017](https://redhat.atlassian.net/browse/HYPERFLEET-1017) | - |
| 4 | **List responses: kind field removed** | Remove `kind` expectations in list response parsing | **Parse error** if client schema requires `kind`. Confirmed: `kind` removed from `ClusterList`, `NodePoolList`, `ResourceList`, `AdapterStatusList`; test asserts `raw.NotTo(HaveKey("kind"))` | Automatable (1178) | [HYPERFLEET-1143](https://redhat.atlassian.net/browse/HYPERFLEET-1143) | - |

### Configuration

| # | Change | Partner Action | Impact if Missed | Classification | Ticket | Parent |
|---|--------|---------------|-----------------|----------------|--------|--------|
| 5 | **Sentinel: messaging_system field removed and config parser now strict** | Remove `messaging_system` from all Sentinel configs and `MESSAGING_SYSTEM` env var. Also remove any other unrecognized fields | **Sentinel fails to start.** v0.2.0 used permissive `v.Unmarshal()`; v1.0.0 uses strict `v.UnmarshalExact()` which rejects unknown fields. Any unrecognized field in the config will cause a startup failure | Automatable (1178) | Sentinel CHANGELOG | - |
| 6 | **Sentinel and Adapter CEL: Ready to Reconciled** | Update ALL `message_decision` CEL in Sentinel AND all precondition/capture CEL in Adapter configs | **CRITICAL SILENT FAILURE.** `condition("Ready")` returns a zero-value struct (empty status, generation 0) via fallback in `decision.go`; CEL evaluates to false; Sentinel never publishes events; adapters never fire; clusters stuck in pending. No error logged | Automatable (1178) | [HYPERFLEET-857](https://redhat.atlassian.net/browse/HYPERFLEET-857) | [HYPERFLEET-559](https://redhat.atlassian.net/browse/HYPERFLEET-559) |
| 7 | **Sentinel Helm: config.hyperfleetApi moved to config.clients.hyperfleetApi** | Move API client config from `config.hyperfleetApi.baseUrl` to `config.clients.hyperfleetApi.baseUrl` in Sentinel Helm values | **Sentinel fails to start.** Helm template references `.Values.config.clients.hyperfleetApi.baseUrl`; old path renders empty; validation fails with "clients.hyperfleet_api.base_url required" | Automatable (1178) | [HYPERFLEET-549](https://redhat.atlassian.net/browse/HYPERFLEET-549), [HYPERFLEET-866](https://redhat.atlassian.net/browse/HYPERFLEET-866) | - |
| 8 | **Sentinel Helm: messageDecision params changed from map to list** | Restructure `messageDecision.params` from map format (`key: 'expr'`) to list format (`- name: key, expr: 'expr'`) | **Helm template fails to render.** Template iterates with `.name` and `.expr` fields; old map format has no such fields. Also fixes non-deterministic param ordering | Automatable (1178) | [HYPERFLEET-1011](https://redhat.atlassian.net/browse/HYPERFLEET-1011) | - |
| 9 | **API Helm: jwt.enabled default changed from true to false** | Explicitly set `config.server.jwt.enabled: true` in Helm values if JWT authentication is required | **API accepts unauthenticated requests.** v0.2.0 Helm chart defaulted to `jwt.enabled: true`; v1.0.0 defaults to `false`. Requests that were previously rejected without a valid JWT token are now accepted | Doc-only (1163/1179) | Helm chart values.yaml diff | - |
| 10 | **JWT identity_claim required when JWT enabled** | Add `server.jwt.identity_claim` pointing to JWT claim for caller identity | **API fails to start** if JWT enabled without identity_claim. Config validation: `if c.IdentityClaim == "" { return fmt.Errorf("server.jwt.identity_claim is required") }`. If claim name is set but does not exist in token, mutating requests (POST/PUT/PATCH/DELETE) fail with auth error; GET requests proceed without identity | Doc-only (1163/1179) | [HYPERFLEET-1134](https://redhat.atlassian.net/browse/HYPERFLEET-1134) | [HYPERFLEET-824](https://redhat.atlassian.net/browse/HYPERFLEET-824) |
| 11 | **Default log format changed to JSON (Sentinel and Adapter)** | Update log parsing pipelines if expecting text format from Sentinel or Adapter | **Log parsing breaks** for Sentinel and Adapter output. `DefaultConfig()` changed from `FormatText` to `FormatJSON` in both components (PRs #103 and #109). API was already JSON in v0.2.0 and is unchanged | Doc-only (1163/1179) | [HYPERFLEET-908](https://redhat.atlassian.net/browse/HYPERFLEET-908) | - |

### Auth

| # | Change | Partner Action | Impact if Missed | Classification | Ticket | Parent |
|---|--------|---------------|-----------------|----------------|--------|--------|
| 12 | **OCM SDK removed; standalone JWT handler** | Remove OCM-specific auth config (`server.acl`, `server.authz`); switch to JWT config (issuer_url, audience, identity_claim) | **Old auth config silently has no effect.** Entire `pkg/client/ocm/` and `pkg/config/ocm.go` removed. Old `server.acl` and `server.authz` Helm values are silently ignored. Combined with #9 (jwt.enabled defaults to false), the API may start with no authentication unless JWT is explicitly configured | Doc-only (1163/1179) | [HYPERFLEET-492](https://redhat.atlassian.net/browse/HYPERFLEET-492) | - |

### Helm Charts

| # | Change | Partner Action | Impact if Missed | Classification | Ticket | Parent |
|---|--------|---------------|-----------------|----------------|--------|--------|
| 13 | **PgBouncer sidecar replaced by generic sidecars** | Rewrite PgBouncer config as generic sidecar entry in `sidecars` list | **No DB proxy sidecar.** Entire `database.pgbouncer.*` Helm values tree removed (not deprecated); old values silently ignored; pod starts without proxy | Doc-only (1163/1179) | [HYPERFLEET-937](https://redhat.atlassian.net/browse/HYPERFLEET-937) | - |
| 14 | **Sentinel config mount paths changed** | Update any custom scripts, init containers, or sidecar configs referencing `/etc/sentinel/config.yaml` or `/etc/sentinel/broker.yaml` to `/etc/hyperfleet/config.yaml` and `/etc/hyperfleet/broker.yaml` | **Config not found.** Custom scripts or containers referencing old path fail | Doc-only (1163/1179) | [HYPERFLEET-549](https://redhat.atlassian.net/browse/HYPERFLEET-549) | - |
| 15 | **Sentinel BROKER_TOPIC env var removed** | Remove `BROKER_TOPIC` env var overrides; use `broker.topic` in Helm values instead | **Topic override ignored.** Env var no longer injected by deployment template; partners overriding topic via env var will have it silently ignored, falling back to Helm value | Doc-only (1163/1179) | [HYPERFLEET-549](https://redhat.atlassian.net/browse/HYPERFLEET-549) | - |
| 16 | **Adapter Helm: broker.type now explicitly required; RabbitMQ fields validated** | Add explicit `broker.type: rabbitmq` or `broker.type: googlepubsub` to adapter Helm values. For RabbitMQ: `url`, `queue`, `exchange`, and `routingKey` are now required | **Helm rendering fails.** `_helpers.tpl` `brokerType` function now uses `required` instead of inference; missing `broker.type` fails at render time. Additionally, `validateBrokerConfig` template requires `url`, `queue`, `exchange`, `routingKey` for RabbitMQ | Automatable (1178) | Adapter Helm chart `_helpers.tpl` | - |

### Database

| # | Change | Partner Action | Impact if Missed | Classification | Ticket | Parent |
|---|--------|---------------|-----------------|----------------|--------|--------|
| 17 | **Fresh database required** | Deploy a completely fresh database. Do not reuse v0.2.0 database | **Policy, not technical limitation.** Migrations technically CAN run on v0.2.0 schema (RENAME COLUMN works), but the project explicitly states "fresh DB, no migration." Reusing old DB risks table locks on rename and untested migration paths | Doc-only (1163/1179) | Epic description | [HYPERFLEET-1176](https://redhat.atlassian.net/browse/HYPERFLEET-1176) |

## Open Actions

1. **Track [HYPERFLEET-1117](https://redhat.atlassian.net/browse/HYPERFLEET-1117):** "API: skip Reconciled and LastKnownReconciled conditions when no required adapters are configured" is in New status. If it merges before v1.0.0, add to this checklist.
