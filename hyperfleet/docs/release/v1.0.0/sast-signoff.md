# Snyk SAST Sign-off — v1.0.0

## Scope

This document records the review and triage of all Snyk Code (SAST) findings for the HyperFleet v1.0.0 release candidate. It covers source-code static analysis only — penetration testing, threat modeling, and container image vulnerability scanning (Clair/Pyxis) are handled separately under the existing Section 4.5 release gate.

SAST scanning is configured in **advisory mode** per [HYPERFLEET-1166](https://redhat.atlassian.net/browse/HYPERFLEET-1166). Findings are not release-gating under the current Enterprise Contract policy (`app-interface-standard`). This sign-off documents the team's proactive review as a supplement to the existing security gates.

### Disposition taxonomy

| Verdict | Meaning |
|---------|---------|
| **Accepted** | Real, reachable capability. Risk consciously accepted with documented rationale — not ignored. |
| **False positive** | Tool incorrectly identified the pattern as a risk. No vulnerability exists. |
| **Noise** | Finding is technically accurate but applies to test code only, with no production security impact. |

### Limitations

Results are from CI-produced SARIF (Konflux pipeline), which applies `--severity-threshold=high` and Red Hat Known False Positive (KFP) filtering. Lower-severity findings that would appear in a local `snyk code test` run may be suppressed. The assurance in this sign-off is bounded by what the CI scan was configured to report.

## Scan metadata

| Field | Value |
|-------|-------|
| Date | 2026-06-16 |
| Tool | Snyk Code (SAST) via `sast-snyk-check-oci-ta` v0.4 (Konflux pipeline) |
| Retrieval | `oras pull` from OCI referrers on `quay.io/redhat-user-workloads/hyperfleet-tenant/hyperfleet/` |
| Scan mode | Advisory (EC policy excludes Snyk SAST rules) |
| Ticket | [HYPERFLEET-1188](https://redhat.atlassian.net/browse/HYPERFLEET-1188) |

### Images scanned

All three commits are the upstream `openshift-hyperfleet/*` main HEAD as of 2026-06-16 (zero drift). v1.0.0 branch has not been cut; main serves as proxy. Sign-off should be re-validated if the release branch diverges.

| Component | Commit | Image digest | Built |
|-----------|--------|-------------|-------|
| hyperfleet-api | `b4bff380` | `sha256:ecffe25d…` | 2026-06-15 |
| hyperfleet-sentinel | `6cb186ed` | `sha256:4e461d86…` | 2026-06-15 |
| hyperfleet-adapter | `34ceb400` | `sha256:00b7e4d3…` | 2026-06-16 |

<details>
<summary>Full identifiers</summary>

| Component | Full commit SHA | Full image digest |
|-----------|----------------|-------------------|
| hyperfleet-api | `b4bff380bd36d56b7f42acfbb416e9c7fa2fc869` | `sha256:ecffe25d562cebde9f08d1ee6f93482e79e66e18751f9402281a9e76d3b29759` |
| hyperfleet-sentinel | `6cb186ed0af3fe9aa6dec71264e603a2a3b2e953` | `sha256:4e461d8647cdd3accd98b3e6e9a271431eb307b1830859692e4123359c1b9af7` |
| hyperfleet-adapter | `34ceb400070fec718249613b91a34e4e05eb7988` | `sha256:00b7e4d3dfef04acec95796d892d3e7dbcb8d4aabf9a8e68588977a9a068c0a3` |

</details>

## Summary

| Component | Total | Accepted | False positive | Noise (test) |
|-----------|-------|----------|---------------|-------------|
| hyperfleet-api | 16 | 0 | 1 | 15 |
| hyperfleet-sentinel | 0 | 0 | 0 | 0 |
| hyperfleet-adapter | 6 | 1 | 0 | 5 |
| **Total** | **22** | **1** | **1** | **20** |

## Findings

### hyperfleet-api

| # | Rule | Sev | Location | CWE | Verdict | Rationale |
|---|------|-----|----------|-----|---------|-----------|
| 1–8 | go/HardcodedPassword/test | note | pkg/config/db_test.go:135,165,214,243,273,289,315,383 | CWE-798 | Noise | Test fixtures using `testpass` — no real credentials |
| 9 | go/NoHardcodedCredentials | note | pkg/config/db.go:88 | CWE-798 | False positive | `NewDatabaseConfig()` sets `Username: "hyperfleet"` as a constructor default. Overridden by env vars, config file, or CLI flags at runtime. Password field is empty string. Not a credential. |
| 10–16 | go/NoHardcodedCredentials/test | note | pkg/config/db_test.go:134,164,213,226,242,272,382 | CWE-798 | Noise | Test fixtures using `testuser` |

### hyperfleet-sentinel

No findings.

### hyperfleet-adapter

| # | Rule | Sev | Location | CWE | Verdict | Rationale |
|---|------|-----|----------|-----|---------|-----------|
| 17 | go/TooPermissiveTrustManager | **warning** | internal/maestroclient/client.go:246 | CWE-295 | **Accepted** | `InsecureSkipVerify = true` is real and reachable, but gated behind `config.Insecure` (line 243), which defaults to `false`. When disabled: HTTPS enforced, custom CA cert loading supported, TLS ≥ 1.2. The insecure path is an explicit operator opt-in via `MAESTRO_INSECURE` env var. Production defaults and Helm values do not set it. Risk is bounded; residual risk lives in operator config. |
| 18–19 | go/HardcodedPassword/test | note | test/integration/maestroclient/setup_test.go:21,123 | CWE-798 | Noise | Test fixture DB passwords |
| 20 | go/TooPermissiveTrustManager/test | note | test/integration/executor/main_test.go:193 | CWE-295 | Noise | Permissive TLS in integration test — expected for envtest self-signed certs |
| 21 | go/TooPermissiveTrustManager/test | note | test/integration/k8sclient/helper_envtest_prebuilt.go:44 | CWE-295 | Noise | Permissive TLS in envtest helper — expected for self-signed certs |
| 22 | go/TooPermissiveTrustManager/test | note | test/integration/maestroclient/setup_test.go:447 | CWE-295 | Noise | Permissive TLS in integration test setup |

## Follow-up items

These are **scan-noise reduction and hardening** tickets, not defect remediation. All findings were triaged above — these tickets improve code hygiene and defense-in-depth.

| Ticket | Type | Component | Scope |
|--------|------|-----------|-------|
| [HYPERFLEET-1233](https://redhat.atlassian.net/browse/HYPERFLEET-1233) | Hardening | hyperfleet-adapter | Finding #17: add startup warning log for insecure TLS mode, enrich `//nolint:gosec` comment with rationale. Findings #18–22: test credential cleanup. |
| [HYPERFLEET-1234](https://redhat.atlassian.net/browse/HYPERFLEET-1234) | Hygiene | hyperfleet-api | Finding #9: remove hardcoded default username from config constructor. Findings #1–8, #10–16: extract test credentials to constants. |

## Sign-off

| | |
|---|---|
| **Reviewer** | Dmitrii Andreev |
| **Date** | 2026-06-16 |
| **Co-sign** | _pending_ |
| **Statement** | All Snyk SAST findings for the v1.0.0 candidate (main branch) have been reviewed across hyperfleet-api, hyperfleet-sentinel, and hyperfleet-adapter. 22 findings triaged: 20 dismissed as test-code noise, 1 false positive, 1 true positive accepted by design (guarded opt-in, strict by default). No unaddressed security-scan defects remain within what the CI scan was configured to report. |
