---
Status: Active
Owner: HyperFleet Team
Last Updated: 2026-06-22
---

# Release Runbook

> **Audience:** Any HyperFleet engineer cutting a release. This is the command sequence — copy, paste, verify. For the broader process (Release Owner duties, communication, sign-offs), read [HyperFleet Release Process](../hyperfleet-release-process.md). For *what happens* between commands, read [Pipeline Anatomy](./pipeline-anatomy.md). For *when things break*, read [Debugging](./debugging.md).

This runbook covers the four real flows:

1. [Cut a release (RC → GA)](#1-cut-a-release-rc--ga)
2. [Fix cycle during RC](#2-fix-cycle-during-rc)
3. [Hotfix after GA](#3-hotfix-after-ga)
4. [Verify what's in Quay](#verification-snippets)

Examples assume Release 1.5 with `hyperfleet-api` going to `v1.5.0`, `hyperfleet-sentinel` to `v1.4.2`, `hyperfleet-adapter` to `v2.0.0`. Substitute your own versions. Independent versioning is intentional — see [Konflux Release Pipeline Design §6](../konflux-release-pipeline-design.md#6-multi-component-configuration).

---

## Preflight

Before you start, confirm:

- [ ] All intended PRs are merged to `main` on each component repo.
- [ ] Nightly E2E has been green for at least 24 hours.
- [ ] You know the target version per component (use existing tags + semver as the guide).
- [ ] You have push access to the component repos and `hyperfleet-release`.
- [ ] You can reach the Konflux UI: <https://konflux-ui.apps.kflux-prd-rh02.0fk9.p1.openshiftapps.com/>.
- [ ] `skopeo` and `jq` are installed locally.

---

## 1. Cut a release (RC → GA)

### 1.1 Create release branches

Per component that has changes since the last release. Components without changes for this release reuse their existing release branch.

```bash
# API: new features → minor bump
cd hyperfleet-api && git checkout main && git pull
git checkout -b release-1.5 && git push origin release-1.5

# Sentinel: bug fixes only → patch bump
cd ../hyperfleet-sentinel && git checkout main && git pull
git checkout -b release-1.4 && git push origin release-1.4

# Adapter: breaking changes → major bump
cd ../hyperfleet-adapter && git checkout main && git pull
git checkout -b release-2.0 && git push origin release-2.0

# Release coordination repo
cd ../hyperfleet-release && git checkout main && git pull
git checkout -b release-1.5 && git push origin release-1.5
```

Nothing happens in Konflux yet — `push.yaml` only triggers on `main`.

### 1.2 Push RC tags per component

```bash
cd hyperfleet-api && git checkout release-1.5
git tag v1.5.0-rc1 && git push origin v1.5.0-rc1

cd ../hyperfleet-sentinel && git checkout release-1.4
git tag v1.4.2-rc1 && git push origin v1.4.2-rc1

cd ../hyperfleet-adapter && git checkout release-2.0
git tag v2.0.0-rc1 && git push origin v2.0.0-rc1
```

Each tag triggers PaC → `tag.yaml` → Konflux build → auto-release. After ~15-20 minutes confirm:

```bash
for svc in api sentinel adapter; do
  case "$svc" in
    api)      tag=1.5.0-rc1 ;;
    sentinel) tag=1.4.2-rc1 ;;
    adapter)  tag=2.0.0-rc1 ;;
  esac
  echo "== hyperfleet-$svc:$tag =="
  skopeo inspect docker://quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/hyperfleet-$svc:$tag \
    | jq -r '.Labels.version'
done
```

If any image is missing or the version label is wrong, see [Debugging: build green but image not in Quay](./debugging.md#build-green-but-image-not-in-quay) or [wrong version baked into the image](./debugging.md#wrong-version-baked-into-the-image).

### 1.3 Update the release manifest and trigger RC E2E

```bash
cd hyperfleet-release && git checkout release-1.5

# Edit RELEASE_MANIFEST.yaml — set per-component versions and e2e_ref
cat > RELEASE_MANIFEST.yaml <<'EOF'
release: "1.5"
e2e_ref: release-1.5
components:
  hyperfleet-api: v1.5.0-rc1
  hyperfleet-sentinel: v1.4.2-rc1
  hyperfleet-adapter: v2.0.0-rc1
EOF

git add RELEASE_MANIFEST.yaml
git commit -m "RC1: API v1.5.0-rc1, Sentinel v1.4.2-rc1, Adapter v2.0.0-rc1"
git push origin release-1.5

# Dry-run the trigger first
./scripts/trigger-rc-e2e.sh --dry-run

# Real run
./scripts/trigger-rc-e2e.sh
```

The script prints the Prow job URL. Watch for tier0 + tier1 pass. If it doesn't start or pulls the wrong tags, see [Debugging: RC E2E didn't trigger](./debugging.md#rc-e2e-didnt-trigger-after-i-tagged-hyperfleet-release).

### 1.4 GA tags

Only after the RC E2E run is green and the Release Owner has signed off. The tag push *is* the gate.

```bash
cd hyperfleet-api && git checkout release-1.5
git tag v1.5.0 && git push origin v1.5.0

cd ../hyperfleet-sentinel && git checkout release-1.4
git tag v1.4.2 && git push origin v1.4.2

cd ../hyperfleet-adapter && git checkout release-2.0
git tag v2.0.0 && git push origin v2.0.0
```

Verify (see [verification snippets](#verification-snippets)).

### 1.5 Finalize the release

```bash
cd hyperfleet-release && git checkout release-1.5

# Update manifest to GA versions
cat > RELEASE_MANIFEST.yaml <<'EOF'
release: "1.5"
e2e_ref: release-1.5
components:
  hyperfleet-api: v1.5.0
  hyperfleet-sentinel: v1.4.2
  hyperfleet-adapter: v2.0.0
EOF

git add RELEASE_MANIFEST.yaml
git commit -m "GA: HyperFleet Release 1.5"
git tag release-1.5
git push origin release-1.5             # push the branch
git push origin tag release-1.5         # push the tag explicitly (same name as branch)
```

Then:

1. Create the GitHub Release on `hyperfleet-release` with notes and the compatibility matrix.
2. Run smoke tests against the GA images.
3. Post to `#hyperfleet-e2e-status` so partner teams pick up.

---

## 2. Fix cycle during RC

Bug found during RC E2E. Only the affected component gets a new RC — others stay pinned.

```bash
# Fix on main first — always
cd hyperfleet-api && git checkout main
# ... open PR with fix + tests, get review, merge ...

# Cherry-pick to the release branch
git checkout release-1.5
git cherry-pick <fix-sha> && git push origin release-1.5

# New RC tag for the affected component only
git tag v1.5.0-rc2 && git push origin v1.5.0-rc2
```

Wait for the build → release run, confirm in Quay (see [verification snippets](#verification-snippets)).

Update the manifest — only the affected component changes:

```bash
cd ../hyperfleet-release && git checkout release-1.5
# Edit RELEASE_MANIFEST.yaml — set hyperfleet-api: v1.5.0-rc2
git add RELEASE_MANIFEST.yaml
git commit -m "RC2: API v1.5.0-rc2 — fix <one-line summary>"
git push origin release-1.5

./scripts/trigger-rc-e2e.sh
```

Repeat until the E2E run is green.

---

## 3. Hotfix after GA

Target: 1 working day for Blocker / Critical CVE. Patches skip the RC cycle — the tag push is the gate, and the focused smoke test is the verification.

```bash
# 1. Fix on main first
cd hyperfleet-api && git checkout main
# ... open PR with fix + tests, get Release Owner review, merge ...

# 2. Cherry-pick to the existing release branch
git checkout release-1.5
git cherry-pick <fix-sha> && git push origin release-1.5

# 3. Push the patch tag — no RC
git tag v1.5.1 && git push origin v1.5.1
```

After ~15-20 minutes:

```bash
skopeo inspect docker://quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/hyperfleet-api:1.5.1 \
  | jq -r '.Labels.version'
```

Then:

1. Smoke test against `…/hyperfleet-api:1.5.1` only.
2. Update `hyperfleet-release/RELEASE_MANIFEST.yaml` — bump the affected component, leave others.
3. Tag `release-1.5.1` on `hyperfleet-release`.
4. Notify in `#hyperfleet-e2e-status` and tag partner teams.

For the broader hotfix process (severity assessment, communication, retrospective triggers), see [HyperFleet Release Process §1.9](../hyperfleet-release-process.md#19-emergency-hotfix-process-post-ga).

---

## Verification snippets

### List all tags for a component

```bash
skopeo list-tags docker://quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/hyperfleet-api \
  | jq -r '.Tags[]' | sort -V | tail -20
```

### Confirm a specific tag and its baked-in version label

```bash
TAG=1.5.0-rc1
skopeo inspect docker://quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/hyperfleet-api:$TAG \
  | jq -r '.Labels.version'
# Should print exactly: 1.5.0-rc1
```

### Spot-check all three components at the same version family

```bash
for svc in api sentinel adapter; do
  for tag in 1.5.0-rc1 1.5.0; do
    echo "hyperfleet-$svc:$tag"
    skopeo inspect docker://quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/hyperfleet-$svc:$tag 2>/dev/null \
      | jq -r '.Labels.version // "MISSING"'
  done
done
```

---

## If something fails

- Tag pushed, no PipelineRun → [Debugging: I pushed a tag and no PipelineRun started](./debugging.md#i-pushed-a-tag-and-no-pipelinerun-started)
- Build red → [Debugging: A build PipelineRun failed](./debugging.md#a-build-pipelinerun-failed)
- Build green but `skopeo` empty → [Debugging: Build green but image not in Quay](./debugging.md#build-green-but-image-not-in-quay)
- Wrong version label → [Debugging: Wrong version baked into the image](./debugging.md#wrong-version-baked-into-the-image)
- RC E2E didn't trigger → [Debugging: RC E2E didn't trigger](./debugging.md#rc-e2e-didnt-trigger-after-i-tagged-hyperfleet-release)
- Anything else → [Support](./support.md)
