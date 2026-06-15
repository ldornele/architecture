---
Status: Active
Owner: HyperFleet Platform Team
Last Updated: 2026-06-16
---

# GCP Developer Cluster Lifecycle Policy

---

## Overview

Defines the lifecycle rules for developer GKE clusters in the `hcm-hyperfleet` GCP project. Developer clusters are personal environments used for development, testing, and experimentation. Without lifecycle controls, these clusters accumulate, leading to orphaned resources and unnecessary costs.

This policy covers:

- Naming and labeling conventions
- TTL and renewal requirements
- Cleanup procedures for orphaned resources
- Event/hackathon cluster rules
- Standalone VM restrictions

---

## Scope

Applies to all GKE clusters in `hcm-hyperfleet` **except**:

- `hyperfleet-dev-prow` (CI/CD, covered by [prow-cicd-cluster.md](prow-cicd-cluster.md))
- Ephemeral `hyperfleet-dev-ci-infra-*` clusters created by Prow for E2E test runs

---

## Cluster Naming and Labeling

### Naming Convention

```text
hyperfleet-dev-{owner}
```

Where `{owner}` is the developer's Kerberos/LDAP username (e.g., `hyperfleet-dev-jsmith`).

### Required Labels

All developer clusters **must** have the following labels:

| Label | Value | Example |
|---|---|---|
| `environment` | `dev` | `dev` |
| `managed-by` | `terraform` | `terraform` |
| `owner` | Developer username | `jsmith` |
| `project` | `hyperfleet` | `hyperfleet` |
| `ttl` | `YYYY-MM-DD` | `2026-06-20` |

The `ttl` label is **mandatory** for all clusters with `environment: dev`. Clusters without a `ttl` label are treated as non-compliant and are automatically shut down.

### Environment Labels and Enforcement

| `environment` value | TTL required | Idle shutdown | Example |
|---|---|---|---|
| `dev` | Yes | Yes | Developer clusters |
| `cicd` | No | No | `hyperfleet-dev-prow` |

---

## Cluster Provisioning

Developer clusters **must** be provisioned via Terraform using the shared module in [hyperfleet-infra](https://github.com/openshift-hyperfleet/hyperfleet-infra). Manual cluster creation via `gcloud` or the console is discouraged.

Requirements:

- Use the `hyperfleet-dev-vpc` network (not `default`)
- Single node pool with `e2-standard-4` (1 node) unless justified
- No autoscaling unless explicitly needed

---

## TTL and Renewal

### Default TTL

Developer clusters have a **5-day TTL** by default. The `ttl` label is set automatically by Terraform at apply time (current date + 5 days). After the TTL date, the cluster is automatically shut down (scaled to zero nodes) by the daily enforcement job.

### Renewal

To renew a cluster, the owner re-applies the Terraform configuration. This resets the `ttl` label to current date + 5 days.

### Expiration

After the TTL date:

1. The daily enforcement job scales the cluster to zero nodes
2. If the TTL is not renewed within **48 hours**, the cluster is deleted on the next enforcement run

---

## Event and Hackathon Clusters

Clusters created for hackathons, workshops, or other time-bound events:

- **Must** include a `ttl=YYYY-MM-DD` label at creation time
- **Must** use the naming convention: `hyperfleet-dev-{event-name}` (e.g., `hyperfleet-dev-hackathon-build`)
- Are automatically flagged for decommissioning after the TTL date
- Should be deleted within **7 days** of the event end date

---

## Standalone VMs

Standalone compute instances (outside of GKE) are **not permitted** in the `hcm-hyperfleet` project. All development workloads must run on GKE clusters provisioned via Terraform.

Exceptions require explicit approval from the team lead.

---

## Automated Daily Enforcement

> **Note:** The automation described below is not yet implemented. See [HYPERFLEET-1229](https://redhat.atlassian.net/browse/HYPERFLEET-1229) for the implementation tracker. Until automation is in place, these rules are enforced manually.

### Idle Node Shutdown

A Cloud Scheduler job runs **every hour** and scales down developer cluster node pools whose nodes have been running for more than **12 hours** (based on node `creationTimestamp`). Developers must manually scale back up when they need the cluster again. This avoids paying for idle compute without requiring a fixed shutdown time, and works equally across all timezones.

Excluded from idle shutdown:

- Clusters with `environment: cicd` (e.g., `hyperfleet-dev-prow`)
- Ephemeral `hyperfleet-dev-ci-infra-*` clusters (managed by Prow, `environment: cicd`)

### Missing Owner Label Enforcement

Clusters **without** a valid `owner` label are treated as unowned and are **automatically shut down** (scaled to zero nodes) on detection. If no owner label is added within **7 days**, the cluster is deleted.

### Future Automation (Proposed)

| Component | Purpose |
|---|---|
| Cloud Scheduler | Hourly cron trigger |
| Cloud Function | Iterates clusters, checks node age (>12h), labels, and TTL; resizes node pools to 0 or deletes |
| Label check | Shuts down clusters missing `owner` label; deletes after 7 days |
| TTL check | Shuts down clusters past their `ttl` date; deletes after 48 hours |
| Terraform update | Add `ttl` label to `common_labels` (current date + 5 days) |

---

## Orphaned Resource Cleanup

### Monthly Audit

A monthly audit should be performed to identify and clean up orphaned resources. The [GCP Asset Inventory dashboard](https://console.cloud.google.com/iam-admin/asset-inventory/dashboard?project=hcm-hyperfleet) provides a consolidated view of all project resources and can help identify leftover assets.

| Resource Type | How to Check | Cleanup |
|---|---|---|
| Unattached PersistentDisks | `gcloud compute disks list` — check for empty `users` field | Delete after confirming no active claim |
| Stale forwarding rules | `gcloud compute forwarding-rules list` — cross-reference with active clusters | Delete if the owning cluster no longer exists |
| Terminated VMs | `gcloud compute instances list` — filter by `TERMINATED` status | Delete VM and associated disks |
| Unused VPC networks | `gcloud compute networks list` — check for networks with no active resources | Delete firewall rules first, then the network |
| Unused static IPs | `gcloud compute addresses list` — check for `RESERVED` status with no users | Release the address |

For automated policy enforcement beyond the daily TTL/shutdown checks, [Cloud Custodian](https://github.com/cloud-custodian/cloud-custodian) is a viable option for defining resource cleanup policies as code.

### Prow E2E Cleanup

Ephemeral `hyperfleet-dev-ci-infra-*` clusters created by Prow should be automatically cleaned up after test completion. If orphaned forwarding rules or disks are found from these clusters, file a bug against the E2E framework.

---

## Audit History

| Date | Auditor | Findings | JIRA |
|---|---|---|---|
| 2026-06-15 | Rafael Benevides | 11 clusters (was 9 at ticket creation), 3 orphaned PVC disks, 3 terminated VMs, 1 unused VPC, 1 orphaned firewall rule. Cleaned up orphans, added TTL to hackathon clusters. | [HYPERFLEET-1112](https://redhat.atlassian.net/browse/HYPERFLEET-1112) |
