---
Status: Active
Owner: HyperFleet Team
Last Updated: 2026-06-22
---

# Debugging: When the Release Pipeline Breaks

> **Audience:** HyperFleet engineers who pushed a tag, merged a PR, or triggered an RC E2E and something didn't happen. Organized by symptom.

Each entry: what you see → where to look → what to check → who to ping. For the happy-path mental model, read [Pipeline Anatomy](./pipeline-anatomy.md) first. For where each config lives, see [Configuration Map](./configuration-map.md).

---

## I pushed a tag and no PipelineRun started

### Where to look

- Konflux UI → Applications → `hyperfleet` → Pipeline runs (filter by component).
- The component repo's GitHub: **Actions** tab is NOT where PaC reports; check the commit status (green check / yellow dot near the commit).

### What to check

1. **The tag actually pushed.** `git push origin v0.3.0-rc1` is one tag; `git push --tags` pushes everything. Verify with `git ls-remote --tags origin | grep v0.3.0-rc1`.
2. **The CEL regex matches your tag.** Open `.tekton/hyperfleet-<svc>-tag.yaml` and look for the `on-cel-expression` annotation. It must match `^refs/tags/v[0-9]+\.[0-9]+\.[0-9]+(-rc[0-9]+)?$`. A tag like `v0.3` or `release-0.3-rc1` won't match.
3. **The PaC GitHub App is installed on the org.** Visit <https://github.com/organizations/openshift-hyperfleet/settings/installations> and confirm "Red Hat Konflux" has access to the component repo.
4. **The Component is wired in `konflux-release-data`.** `tenants-config/cluster/kflux-prd-rh02/tenants/hyperfleet-tenant/applications/hyperfleet/components/<svc>.yaml` should have `build.appstudio.openshift.io/request: configure-pac`.

**Who to ping:** PaC install issues — `#forum-konflux-release` on Red Hat Slack.

---

## A build PipelineRun failed

### Where to look

- The failing task in the Konflux UI → **Logs** tab.

### What to check

1. **Is it advisory or blocking?** See the task table in [Pipeline Anatomy](./pipeline-anatomy.md#the-build-dag). Under `app-interface-standard`, scan tasks (Snyk, Clair, ClamAV, shell/unicode SAST) are advisory — they surface findings but don't block the release. A red badge on `sast-snyk-check` does **not** stop the build pipeline from being marked failed in the UI even when the EC verdict treats it as advisory — read the actual error.
2. **`build-container` failed → Dockerfile issue.** Local reproduce: `buildah build --build-arg APP_VERSION=0.3.0-rc1 .` from a clean checkout of the tag.
3. **`extract-version` failed.** The task expects `target_branch` to start with `refs/tags/v`. A non-tag trigger or a malformed tag breaks it.
4. **`prefetch-dependencies` failed.** Cachi2 — non-hermetic today, so most failures are transient. Re-run the PipelineRun from the UI.
5. **Snyk SARIF inspection.** Pull the SARIF from the task results; the UI columns mislead. See the spike note `Verifying Snyk SAST Outputs` for the skopeo-based extraction.

**Who to ping:** Konflux build infra — `#konflux-users`. EC policy questions — `#forum-conforma`.

---

## Build green but image not in Quay

The classic. The build PipelineRun and the release PipelineRun are separate. Build green ≠ image pushed.

### Where to look

- Konflux UI → **Releases** tab (NOT pipeline runs). There should be a Release CR created from the Snapshot.
- If the Release exists but is incomplete, click into it and inspect the managed PipelineRun (`rh-push-to-external-registry`).

### What to check

1. **Snapshot created.** Build run → Snapshots → the Snapshot for this commit.
2. **EC verdict.** Snapshot → Enterprise Contract result. Failed EC → release will not auto-fire. Read the violation messages; if they're advisory-tagged, they shouldn't block (`app-interface-standard` excludes most).
3. **Release CR exists.** If no Release was created, the Snapshot's `auto-release` is off — check the ReleasePlan label `release.appstudio.openshift.io/auto-release: 'true'`.
4. **`rh-push-to-external-registry` ran and failed.** Open the managed PipelineRun; most failures are missing secrets (see next section).
5. **`skopeo list-tags` empty 60s after release run finishes.** Almost always a Quay propagation delay — wait two minutes and retry.

**Who to ping:** RPA / managed pipeline behavior — `#forum-konflux-release`.

---

## Release pipeline failed

Once the build Snapshot passes EC, `rh-push-to-external-registry` runs. The common failures:

| Symptom in the managed PipelineRun | Likely cause | Fix |
|------------------------------------|--------------|-----|
| `verify-access` task failed | RPA / ReleasePlan mismatch, or the service account lost Quay push access | Check `serviceAccountName: release-app-interface-prod` in the RPA matches what's provisioned. Ping RelEng. |
| `collect-data` task failed | RPA `data.mapping` references a component not in the Snapshot | Confirm the Application's Components and the RPA mapping list are aligned |
| `push-snapshot` task failed | Quay push auth failed (`konflux-release-service-access-management-token` rotated) | RelEng — `#forum-konflux-release` |
| `create-pyxis-image` task failed | Pyxis secret missing or expired in `rhtap-releng-tenant` | RelEng |
| `slack-notification` task failed | `hyperfleet-slack-webhook-notification-secret` missing, or webhook URL revoked | See [Notifications](./notifications.md#rotating-the-slack-webhook). The release itself usually still completes — the notification is in the `finally` block. |
| EC violation in the release run (different from build EC) | Policy `app-interface-standard` constraint we don't meet | Inspect; if the violation is real-engineering, fix. If it's an exemption candidate, file an exception in `konflux-release-data/exceptions/`. |

---

## Wrong version baked into the image

Symptom: `skopeo inspect …:0.3.0-rc1 | jq -r '.Labels.version'` returns `0.0.0-dev` or empty.

### Where to look

- The build run's `extract-version` task → **Logs**.
- The build run's `build-container` task → **Logs** (look for `--build-arg APP_VERSION=…`).

### What to check

1. The `extract-version` output is the version string with the `v` stripped. If it printed `0.0.0-dev`, the trigger was a `push` event on main, not a tag — the run name will be `…-on-push-…`. The build did what it was told.
2. If the trigger was the tag and `extract-version` printed correctly but the LABEL is `0.0.0-dev`: the Dockerfile lost its `ARG APP_VERSION` or its `LABEL version="${APP_VERSION}"`. Re-check the Dockerfile contract — see [Configuration Map: component repos](./configuration-map.md#component-repos-hyperfleet-api-hyperfleet-sentinel-hyperfleet-adapter).
3. Multi-stage Dockerfiles: `ARG APP_VERSION` must be **redeclared in every stage** that references it. Missing redeclaration is the #1 cause.

---

## RC E2E didn't trigger after I tagged hyperfleet-release

### Where to look

- `hyperfleet-release/scripts/trigger-rc-e2e.sh` — run with `--dry-run` first.
- The Gangway HTTP response from the script's log output.
- The Prow dashboard for our job.

### What to check

1. **Manifest images exist in Quay.** The script verifies each component image at the version pinned in `RELEASE_MANIFEST.yaml`. If any image is missing, the script bails before calling Gangway. `skopeo list-tags` to confirm.
2. **`oc` token from app.ci is fresh.** The Gangway call needs a token; if it 401s, refresh from <https://oauth-openshift.apps.ci.l2s4.p1.openshiftapps.com/oauth/token/request>.
3. **Job config drift in `openshift/release`.** If the periodic / triggered job name doesn't match what the script sends, Gangway accepts the call but no job appears. Compare against the latest ci-operator config under `ci-operator/config/openshift-hyperfleet/`.
4. **GitHub Action not yet wired.** The interim workflow is manual via the script — there's no auto-trigger on tag push yet (see [HYPERFLEET-1038](https://redhat.atlassian.net/browse/HYPERFLEET-1038)).

For full mechanics see [Trigger HyperFleet E2E Jobs via Gangway API](../test-release/trigger-e2e-jobs-via-gangway.md).

---

## RC E2E ran but pulled the wrong image tags

### Where to look

- The Prow job → environment variables → `*_IMAGE_TAG`, `*_IMAGE_REPO`.

### What to check

1. **`RELEASE_MANIFEST.yaml` was current when the script ran.** The script reads the file at invocation time; if you committed the manifest after pushing the tag, the previous version was used.
2. **The script strips the `v` prefix.** Quay tags have no `v`; the manifest entries do. The script handles the conversion — if you bypass the script and call Gangway directly, you have to do this yourself.
3. **Manifest typo.** Component keys must be `hyperfleet-api`, `hyperfleet-sentinel`, `hyperfleet-adapter` exactly.

---

## Enterprise Contract violation

EC runs in two places: in the build PipelineRun's `verify-enterprise-contract` task, and again in the release pipeline. The build-side verdict gates the Snapshot; the release-side verdict gates the push.

### What to check

1. **Policy applied.** Container RPA uses `app-interface-standard`; chart RPA uses `registry-hyperfleet-chart-prod`. Mismatch → wrong rules applied → spurious failures.
2. **RPA `origin` matches the tenant.** `origin: hyperfleet-tenant`. Required for the constraint to bind.
3. **Constraint regex.** `constraints/service/hyperfleet.yaml` enforces the Quay URL prefix. A typo in the RPA's `mapping.components[].repositories[].url` will trip the constraint at MR-validation time (`tox -e test`), not at runtime.
4. **Genuine policy gap.** If the violation is real and unfixable in the short term, file an exception under `konflux-release-data/exceptions/` with rationale.

---

## Escalation

- General Konflux platform — `#konflux-users` (Red Hat Slack)
- Release pipeline (RPA, managed pipelines) — `#forum-konflux-release`
- Enterprise Contract — `#forum-conforma`
- Hyperfleet release coordination — `#hyperfleet-e2e-status`, then page the Release Owner
- File a Konflux support ticket: project `KFLUXSPRT` on JIRA

See [Support](./support.md) for the full list with links.
