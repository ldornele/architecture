---
Status: Active
Owner: HyperFleet Team
Last Updated: 2026-06-22
---

# Support and Escalation

> **Audience:** HyperFleet engineers who need to find the right channel, dashboard, or contact fast. No prose — just tables.

For *why* something is configured a particular way, see [Konflux Release Pipeline Design](../konflux-release-pipeline-design.md). This page is a phone book.

---

## Slack channels

| Channel | When to use it |
|---------|----------------|
| `#hyperfleet-e2e-status` | Watch HyperFleet release notifications. Coordinate releases here. |
| `#konflux-users` | General Konflux platform questions, build infra issues |
| `#forum-konflux-release` | RPA, managed pipelines, release service behavior |
| `#forum-conforma` | Enterprise Contract policy questions, exceptions |

All on Red Hat enterprise Slack.

---

## JIRA queues

| Project | Purpose | URL |
|---------|---------|-----|
| `HYPERFLEET` | Our own tickets | <https://redhat.atlassian.net/browse/HYPERFLEET> |
| `KFLUXSPRT` | Konflux support tickets (file when the platform team needs to act) | <https://issues.redhat.com/browse/KFLUXSPRT> |
| `KFLUXMIG` | Konflux migration tracker (per-tenant onboarding tracking) | <https://issues.redhat.com/browse/KFLUXMIG> |

---

## Konflux UI and source repos

| Resource | URL |
|----------|-----|
| Konflux UI (`kflux-prd-rh02`) | <https://konflux-ui.apps.kflux-prd-rh02.0fk9.p1.openshiftapps.com/> |
| `konflux-release-data` (GitLab) | <https://gitlab.cee.redhat.com/releng/konflux-release-data> |
| Release service catalog (pipelines) | <https://github.com/konflux-ci/release-service-catalog> |
| Konflux docs | <https://konflux-ci.dev/docs/> |
| App Interface onboarding walkthrough | <https://konflux.pages.redhat.com/docs/users/end-to-end/onboard-containers-for-app-interface.html> |
| Enterprise Contract docs | <https://enterprisecontract.dev/> |
| EC policy source | <https://github.com/release-engineering/rhtap-ec-policy> |
| Pipelines as Code docs | <https://pipelinesascode.com/docs/> |

---

## Prow

| Resource | URL |
|----------|-----|
| Prow dashboard (`app.ci`) | <https://prow.ci.openshift.org/> |
| `openshift/release` repo | <https://github.com/openshift/release> |
| Gangway token request | <https://oauth-openshift.apps.ci.l2s4.p1.openshiftapps.com/oauth/token/request> |

---

## Access management

| Need | How to get it |
|------|---------------|
| `hyperfleet-tenant` access on `kflux-prd-rh02` | Join the team Rover group — see <https://rover.redhat.com/groups/> |
| `konflux-release-data-users` (GitLab) | Self-service join on GitLab |
| `openshift-hyperfleet` GitHub team | Org admin grants |
| `rhtap-releng-tenant` secrets (Slack, Pyxis) | File a ticket with RelEng in `#forum-konflux-release` — HyperFleet does not have direct write access |

---

## Escalation order

1. **Search the docs in this directory first** — most operational answers are here.
2. **`#hyperfleet-e2e-status` or the team channel** — for HyperFleet-specific coordination.
3. **Forum channels** above — for platform-side issues. Tag your message with what you've already checked.
4. **File a `KFLUXSPRT` ticket** — when the platform team needs to act and Slack isn't moving fast enough.
5. **Page the Release Owner** — for in-flight release blockers. See the release process doc for the on-call rotation.

---

## Related

- [Configuration Map](./configuration-map.md) — where each file mentioned in support conversations lives
- [Debugging](./debugging.md) — try this before pinging
- [HyperFleet Release Process](../hyperfleet-release-process.md) — Release Owner role and the broader process
