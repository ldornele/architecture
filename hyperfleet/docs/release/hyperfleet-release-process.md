---
Status: Active
Owner: HyperFleet Team
Last Updated: 2026-04-21
---

# HyperFleet Release Process

---

## Overview

This documentation defines the release process for HyperFleet (hyperfleet-api, hyperfleet-sentinel, and hyperfleet-adapter). Container images are built and released via Konflux; Prow handles PR validation and E2E testing. See [Konflux Release Pipeline Design](./konflux-release-pipeline-design.md) for the pipeline architecture and [ADR 0014](../../adrs/0014-konflux-build-and-release.md) for the decision rationale.

**Key Recommendations:**

- **Hybrid release cadence:** Regular releases (TBD by Tech Lead) for quality + ad-hoc releases for urgent requirements
  - Weeks 1-2 (TBD by Tech Lead): Active development
  - End of Week 2 (TBD by Tech Lead): Release Readiness Review (go/no-go)
  - Week 3 (TBD by Tech Lead, estimated 5 working days): Feature Freeze → Stabilization → Code Freeze → GA Release
- Git branching strategy with release branches (fix main first → cherry-pick to release branch)
- **Independent component versioning:** Each component (hyperfleet-api, hyperfleet-sentinel, hyperfleet-adapter) maintains its own semantic version
- Validated HyperFleet releases defined by compatibility-tested component version combinations
- Multi-gate release readiness criteria including automated and manual validation
- Structured bug triage workflow post-code freeze
- Comprehensive release artifacts including container images, Helm charts, and documentation
- **Konflux builds and releases container images**; Prow retains PR validation and E2E testing (see [Konflux Release Pipeline Design](./konflux-release-pipeline-design.md))

---

## 1. Release Owner Checklist

This checklist guides the Release Owner through each phase of the release process. Use it to track progress, ensure all criteria are met, and communicate status to stakeholders.

### 1.1 Stakeholder Communication

**Key Stakeholders:**

- Development Team
- QE Owner
- CI Owner (Prow/Konflux)
- Tech Lead
- Manager

**Primary Communication Channels:**

- Slack: `#hyperfleet-releases` (create if not exists)
- Release ticket created by Tech Lead during sprint planning

---

### 1.2 Phase 1: Pre-Release Planning

**Timing:** End of Week 2, 1-2 days before Feature Freeze (TBD by Tech Lead)

This is your **Release Readiness Review** - assess whether you're ready to cut the release branch.

**Note on Release Owner Selection:**
The Release Owner for each release is identified before the release process begins. Initially, this role will be filled by the release document drafter. In the future, this will follow a rotation mechanism where team members take turns serving as Release Owner.

#### Setup

- [ ] Confirm Release Owner assignment in release ticket
- [ ] Identify target HyperFleet Release number (e.g., Release 1.5)
- [ ] Determine component versions and branching strategy (see [Versioning Strategy](#35-versioning-strategy))

#### Feature Completeness Assessment

- [ ] Review all planned features for milestone - which are code-complete? (see [Feature Completeness Gate](#21-feature-completeness-gate))
- [ ] Identify features that won't make this release (defer to next)
- [ ] Feature toggles in place for incomplete features if applicable
- [ ] Feature documentation drafted for completed features
- [ ] No CRITICAL/HIGH security vulnerabilities unaddressed (once konflux job is supported)
- [ ] Technical debt reviewed and acceptable items explicitly deferred to next release
- [ ] All deprecated APIs have migration paths documented

#### Documentation Readiness

- [ ] Release notes draft exists with completed features
- [ ] Known issues documented
- [ ] Component documentation up-to-date:
  - [ ] hyperfleet-api documentation
  - [ ] hyperfleet-sentinel documentation
  - [ ] hyperfleet-adapter documentation
- [ ] Compatibility matrix documented showing which component versions work together
- [ ] Breaking changes (if any) documented with migration guides and version requirements

#### CI/CD and Build Health

- [ ] Prow CI pipeline is green for all components on the main branch
- [ ] Container images build successfully for all target architectures
- [ ] Helm charts package without errors

#### Stakeholder Communication

- [ ] **Slack**: Announce release readiness status to `#hyperfleet-releases`

  ```text
  📅 HyperFleet Release X.Y - Readiness Review
  - Feature Freeze: [DATE] (Week 3, Day 1)
  - Target GA: [DATE] (Week 3, Day 5)
  - Release Owner: @your-name
  - Status: [Ready/At Risk - explain]
  ```

- [ ] **Tech Lead**: Confirm feature scope
- [ ] **CI Owner**: Verify Prow jobs are stable on main branch
- [ ] **QE Owner**: Confirm testing capacity for Week 3

#### Go/No-Go Decision

- [ ] **If READY**: Proceed to Feature Freeze on Week 3, Day 1
- [ ] **If AT RISK**:
  - [ ] Identify specific blockers and estimated resolution time
  - [ ] Raise risk to **Manager** and **Tech Lead** for approval
  - [ ] Options: Defer release by X days, or descope features and proceed

**Exit Criteria:** All features complete, documentation drafted, CI green, stakeholder alignment → Proceed to Feature Freeze

---

### 1.3 Phase 2: Feature Freeze

**Timing:** Start of Week 3 in sprint (TBD by Tech Lead)

#### Branch Creation

Create branches for components and supporting repositories (see [Branching Model](#31-branching-model)):

```bash
# Example: hyperfleet-api getting v1.5.0
cd openshift-hyperfleet/hyperfleet-api
git checkout main && git pull origin main
git checkout -b release-1.5
git push origin release-1.5
```

For components (using component-specific semantic version):

- [ ] **hyperfleet-api**: Create `release-X.Y` branch (if version bump)
- [ ] **hyperfleet-sentinel**: Create `release-X.Y` branch (if version bump)
- [ ] **hyperfleet-adapter**: Create `release-X.Y` branch (if version bump)

For supporting repos (using HyperFleet release version, e.g., `release-1.5`):

- [ ] **hyperfleet-e2e**: Create `release-X.Y` branch
- [ ] **hyperfleet-infra**: Create `release-X.Y` branch and update charts, images to correct component versions
- [ ] **hyperfleet-release**: Create `release-X.Y` branch

#### Verify Pipeline Readiness

**Note:** Konflux handles release branch builds automatically via PaC tag-triggered pipelines — no per-branch job configuration needed. See [Konflux Release Pipeline Design](./konflux-release-pipeline-design.md).

- [ ] Verify `.tekton/` pipeline files exist in each component repo
- [ ] Verify RC E2E Prow job is configured (single reusable job, triggered via Gangway)

#### Cut First Release Candidate

For each component, tag RC1 (see [Code Freeze Mechanics](#33-code-freeze-mechanics)):

```bash
# Example: hyperfleet-api v1.5.0-rc1
git checkout release-1.5
git tag -a v1.5.0-rc1 -m "hyperfleet-api RC1 for v1.5.0"
git push origin v1.5.0-rc1
```

- [ ] Tag RC1 for each component: `vX.Y.0-rc1`
- [ ] **Verify Konflux builds container images for all RC1 tags** (wait for builds to complete)
- [ ] Confirm all RC1 images pushed to `quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/hyperfleet-*`

#### Stakeholder Communication

- [ ] **Slack**: Announce Feature Freeze

  ```text
  🔒 Feature Freeze - HyperFleet Release X.Y
  - Release branches created
  - RC1 tagged: [component versions]
  - Main branch OPEN for X.Y+1 development
  - Bug fixes: Fix on main first, then cherry-pick to release branch
  ```

- [ ] **Developers**: Notify team that `main` branch reopens for X.Y+1 development
- [ ] **QE Owner**: Notify RC1 ready for testing, share component versions

**Exit Criteria:** Release branches created, RC1 tagged, Prow jobs configured → Begin Stabilization

---

### 1.4 Phase 3: Stabilization & Testing

**Timing:** Days 1-3 of Week 3 (TBD by Tech Lead)

#### Testing Execution (see [Testing & Validation](#41-testing--validation-mandatory))

- [ ] **QE Owner**: E2E test suite execution started
- [ ] **QE Owner**: Cross-component compatibility validation
- [ ] **QE Owner**: Backward compatibility testing (N-1 version)
- [ ] **QE Owner**: Performance benchmarks (no regression > 10%) (once performance testing is supported)

#### Bug Triage (see [Bug Triage Process](#51-bug-triage-process))

Monitor bugs reported during stabilization:

- [ ] Review all new bugs daily with severity assignment
- [ ] **Blocker/Critical bugs**: Assign developer immediately (see [Decision Framework](#52-decision-framework))
  - [ ] Developer fixes on main first (link bug ticket in PR)
  - [ ] Cherry-pick to release branch (same bug ticket, stays open until both merged)
  - [ ] Both developer AND Release Owner verify cherry-pick completion
  - [ ] Cut new RC if needed (e.g., `vX.Y.0-rc2`)
  - [ ] Re-run full E2E test suite to validate RC and detect regressions
- [ ] **Major bugs**: Evaluate fix or defer decision (see [Decision Framework](#52-decision-framework))
- [ ] **Normal/Minor bugs**: Defer to next release

#### Bug Fix Process (see [Bug Fix Workflow](#33-code-freeze-mechanics))

For each bug fix needed in release:

```bash
# 1. Fix on main first
git checkout main
# Create PR to main (link bug ticket) → merge with tests

# 2. Cherry-pick to release branch
git checkout release-X.Y
git cherry-pick <fix-sha>
# Create PR to release-X.Y (link SAME bug ticket) → Release Owner approves

# 3. Verification by BOTH developer and Release Owner
# Bug ticket stays open until BOTH PRs merged
```

- [ ] All release PRs have Release Owner approval
- [ ] All release PRs include justification and risk assessment
- [ ] All bug tickets linked in both main and release PRs
- [ ] Maintain cherry-pick tracking list (check daily for pending cherry-picks)
- [ ] Verify cherry-pick completion before closing bug tickets

#### Stakeholder Communication

- [ ] **Daily**: Update release ticket with bug triage status
- [ ] **Daily Slack Update** (`#hyperfleet-releases`): Post daily status update

  ```text
  📊 Daily Status - HyperFleet Release X.Y (Day N of Stabilization)
  - Testing Progress: [E2E: 80% complete, Integration: PASSING]
  - Bugs Found: [X Blocker, Y Critical, Z Major]
  - Risks: [Any delays, blockers, or concerns]
  - ETA: [On track / At risk - explain]
  ```

- [ ] **When new RC cut**: Notify QE Owner that RC is ready for testing, share updated component versions
- [ ] **Tech Lead**: Review major bugs and defer/fix decisions

**Exit Criteria:** E2E tests passing, no Blocker/Critical/Major bugs → Enter Code Freeze

---

### 1.5 Phase 4: Code Freeze

**Timing:** Days 4-5 of Week 3 (TBD by Tech Lead)

#### Code Freeze Gate

- [ ] **Mandatory**: No open Blocker/Critical/Major bugs (see [Bug Severity Gates](#42-bug-severity-gates-mandatory))
- [ ] **Mandatory**: E2E tests passing
- [ ] **Mandatory**: Cross-component compatibility validated

#### Stakeholder Communication

- [ ] **Slack**: Announce Code Freeze

  ```text
  ❄️ Code Freeze - HyperFleet Release X.Y
  - Only CRITICAL fixes allowed
  - All PRs require Release Owner approval
  - GA target: [DATE]
  - Outstanding items: [list or "none"]
  ```

- [ ] **When new RC cut**: Notify QE Owner that RC is ready for testing, share updated component versions

#### Final Validation (see [Release Readiness Criteria](#4-release-readiness-criteria))

- [ ] All unit/integration tests passing (see [Testing & Validation](#41-testing--validation-mandatory))
- [ ] E2E critical user workflows validated
- [ ] Performance benchmarks within acceptable bounds (once performance testing is supported)
- [ ] Installation/upgrade path tested
- [ ] Vulnerability scanning: No CRITICAL/HIGH CVEs (see [Security & Compliance](#45-security--compliance))

#### Documentation Finalization (see [Documentation Completeness](#43-documentation-completeness-mandatory))

- [ ] Release notes finalized (see [Release Notes](#721-release-notes))
  - [ ] What's New section complete
  - [ ] Breaking changes documented
  - [ ] Known issues listed
  - [ ] Compatibility matrix complete
- [ ] Upgrade guide finalized (see [Upgrade/Installation Guide](#722-upgradeinstallation-guide))
- [ ] Component documentation updated (see [Component Documentation](#723-component-documentation))
  - [ ] hyperfleet-api documentation
  - [ ] hyperfleet-sentinel documentation
  - [ ] hyperfleet-adapter documentation

#### Critical Fix Approval (see [Post-Code Freeze PR Approval](#53-post-code-freeze-pr-approval-process))

If critical fix needed during code freeze:

- [ ] PR includes severity justification
- [ ] PR includes risk assessment
- [ ] Minimum 2 approvals (reviewer + Release Owner)
- [ ] Prow tests green
- [ ] Cut new RC: `vX.Y.0-rcN`
- [ ] Re-run full E2E test suite to validate RC and detect regressions

#### Final Stakeholder Sign-Off

- [ ] **QE Owner**: Confirm final test results
- [ ] **CI Owner**: Confirm Prow pipeline health
- [ ] **Tech Lead**: Review and approve GA readiness
- [ ] **Pillar Teams**: Notify for integration validation (if applicable)

**Exit Criteria:** All release readiness criteria met (see [Release Readiness Criteria](#4-release-readiness-criteria)) → Proceed to GA Release

---

### 1.6 Phase 5: GA Release

**Timing:** End of Day 5, Week 3 (TBD by Tech Lead)

#### Tag GA Release

For each component, tag final version (see [Practical Example](#352-practical-example-hyperfleet-release-15)):

```bash
# Example: hyperfleet-api v1.5.0 GA
cd openshift-hyperfleet/hyperfleet-api
git checkout release-1.5
git tag -a v1.5.0 -m "hyperfleet-api v1.5.0 - GitOps integration"
git push origin v1.5.0
```

- [ ] Tag GA for each component: `vX.Y.Z`
- [ ] Verify Konflux builds and pushes container images to registry

#### Release Artifacts (see [Release Artifacts](#7-release-artifacts-and-deliverables))

- [ ] Container images published to `quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/hyperfleet-*`
- [ ] Helm charts packaged and tested
- [ ] Git tags created with correct versions
- [ ] Release notes published in `hyperfleet-release` repo
- [ ] Compatibility matrix documented

#### Publish GitHub Release

- [ ] In `hyperfleet-release` repo `release-X.Y` branch (created at Feature Freeze), finalize release notes and compatibility matrix
- [ ] Create GitHub Release entry with title "HyperFleet Release X.Y"
- [ ] Release notes body: Include what's new, breaking changes, known issues, and full compatibility matrix
- [ ] Attach artifacts (if applicable)

#### Stakeholder Communication

- [ ] **Slack**: Announce GA Release

  ```text
  🚀 HyperFleet Release X.Y - GA
  - hyperfleet-api: vX.Y.Z
  - hyperfleet-sentinel: vX.Y.Z
  - hyperfleet-adapter: vX.Y.Z
  - Release Notes: [LINK]
  - Container Images: quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/hyperfleet-*
  ```

- [ ] **Developers**: Notify release is complete
- [ ] **Offering Team**: Notify for GCP integration deployment
- [ ] **Pillar Teams**: Share release notes and upgrade instructions

**Exit Criteria:** All artifacts published, stakeholders notified → Enter Post-Release

---

### 1.7 Phase 6: Post-Release

**Timing:** Immediately after GA

#### Immediate Actions (Day 1)

- [ ] Monitor nightly Prow jobs on release branch
- [ ] Monitor for critical bugs reported
- [ ] Update release ticket status to "Completed"

#### Retrospective (Within 1 week) if required

- [ ] Schedule retrospective with team (see [Conduct Retrospectives and Identify Improvements](#911-conduct-retrospectives-and-identify-improvements))
- [ ] Collect metrics:
  - [ ] Code freeze duration
  - [ ] Number of RCs cut
  - [ ] Bugs found post-Feature Freeze
  - [ ] On-time delivery (yes/no)
- [ ] Document lessons learned
- [ ] Update release process if needed

#### Patch Release Monitoring (see [Release Branch Maintenance](#34-release-branch-maintenance))

- [ ] Monitor nightly Prow E2E jobs on `release-X.Y` branch (runs until EOL at 6 months)
- [ ] Investigate and address any nightly test failures to detect regressions early
- [ ] Track bugs for potential patch releases
- [ ] Blocker/Critical CVE → Hotfix within 1 working day (see [Hotfix Workflow](#54-hotfix-workflow-post-ga))
- [ ] All other bugs → Patch within 2 working days, or defer to next release
- [ ] Disable nightly jobs when release reaches EOL

#### Stakeholder Communication if required

- [ ] **Slack**: Share retrospective findings
- [ ] **Tech Lead**: Discuss process improvements

---

### 1.8 Ad-Hoc Release Process

When urgent release needed outside regular cadence:

#### Request Evaluation

- [ ] Create ad-hoc release request using template (Appendix C)
- [ ] Evaluate justification: Why can't this wait?
- [ ] Assess risk level and blast radius
- [ ] Determine if Full HyperFleet Release or Single Component Patch

#### Approval

- [ ] Get Tech Lead approval
- [ ] Get Manager approval
- [ ] Document approval and conditions in request issue

#### Execution (3-5 days timeline)

- [ ] Follow condensed testing plan (unit, integration, E2E for affected components)
- [ ] Cut RC and validate
- [ ] Tag GA release
- [ ] Notify stakeholders 2 working days in advance minimum
- [ ] Monitor post-release for 1 working day

---

### 1.9 Emergency Hotfix Process (Post-GA)

For critical bugs discovered after GA:

#### Severity Assessment

- [ ] Confirm severity: Blocker or Critical only
- [ ] Identify affected component(s)
- [ ] Estimate impact and timeline

#### Hotfix Execution (see [Hotfix Workflow](#54-hotfix-workflow-post-ga))

```bash
# 1. Fix on main first
git checkout main
# Create PR with fix + tests, link bug ticket, merge after review

# 2. Cherry-pick to release branch
git checkout release-X.Y
git cherry-pick <fix-sha>
git push origin release-X.Y

# 3. Tag patch release (no RC needed for patches)
git tag -a vX.Y.Z -m "Patch release vX.Y.Z"
git push origin vX.Y.Z
# → PaC tag.yaml triggers → Konflux builds → auto-release to Quay
```

- [ ] Fix on main first (link bug ticket in PR)
- [ ] Cherry-pick to release branch (same bug ticket, stays open until both merged)
- [ ] Both developer AND Release Owner verify cherry-pick completion
- [ ] Tag next patch version: `vX.Y.Z` (NO RC for patches, see [Branching Model](#31-branching-model))
- [ ] Run focused test suite
- [ ] Deploy hotfix

#### Timeline

- [ ] **Blocker/Critical CVE**: Patch within 1 working day

Non-critical issues do not use the emergency hotfix process — they follow the standard patch release workflow (2 working days) or are deferred to the next release.

#### Stakeholder Communication

- [ ] **Slack**: Immediate notification of critical issue

  ```text
  🚨 Critical Hotfix - Component vX.Y.Z
  - Issue: [description]
  - Severity: Critical
  - ETA: [timeline]
  - Tracking: [LINK]
  ```

- [ ] **Slack**: Notify when hotfix released
- [ ] **Offering Team**: Coordinate deployment

---

## 2. Release Entry Criteria

These criteria determine when the development team can initiate the release process and enter code freeze.

### 2.1 Feature Completeness Gate

**Decision Point:** Can we enter Feature Freeze and create release branches?

**Gate Criteria:**

- ✓ All planned features code-complete OR explicitly deferred with justification
- ✓ No CRITICAL/HIGH security vulnerabilities unaddressed
- ✓ Feature documentation exists for completed features
- ✓ Technical debt reviewed and deferred items documented
- ✓ Deprecated APIs have migration paths

**If criteria not met:**

- **Option 1:** Defer release to allow feature completion (requires Tech Lead approval)
- **Option 2:** Descope incomplete features and proceed on schedule
- **Option 3:** Accept risk with documented justification (Manager approval required)

**Detailed execution checklist:** See [Pre-Release Planning - Feature Completeness Assessment](#12-phase-1-pre-release-planning)

### 2.2 Testing & Quality Gate

**Decision Point:** Is the codebase stable enough to proceed with Feature Freeze?

**Gate Criteria:**

- ✓ CI/CD pipeline green on main branch (unit, integration, E2E tests passing)
- ✓ Container images build successfully for all target architectures
- ✓ Helm charts package without errors
- ✓ No performance regression >10% vs. previous release (once performance testing is supported)

**If criteria not met:**

- **Blocker:** Fix failing tests before Feature Freeze (CI must be green)
- **Build failures:** Resolve build issues immediately (blocks release branch creation)
- **Performance regression:** Evaluate impact - fix or defer with Tech Lead approval

**Quality Standards:**

- Unit test coverage ensured by pre-submit jobs (not part of release gating)
- Integration and E2E tests validate critical user journeys
- Performance benchmarks track regression trends

**Detailed execution checklist:** See [Pre-Release Planning - CI/CD and Build Health](#12-phase-1-pre-release-planning) and [Stabilization & Testing](#14-phase-3-stabilization--testing)

### 2.3 Cross-Component Compatibility Gate

**Decision Point:** Are component versions determined and integration validated?

**Gate Criteria:**

- ✓ Component versions determined using independent semantic versioning
- ✓ Cross-component API contracts validated during integration testing
- ✓ Component version combinations pass integration tests

**Versioning Model:**

- Each component (hyperfleet-api, hyperfleet-sentinel, hyperfleet-adapter) maintains its own semantic version
- HyperFleet Release X.Y defines a validated, compatibility-tested set of component versions
- See [Versioning Strategy](#35-versioning-strategy) for detailed approach

**If criteria not met:**

- **Compatibility untested:** Execute integration test suite before proceeding to Feature Freeze
- **API contract violations:** Fix compatibility issues or update API contracts with proper versioning

**Note:** Full backward compatibility (N-1 upgrade) testing happens during Stabilization phase (see [Stabilization & Testing](#14-phase-3-stabilization--testing))

**Detailed execution checklist:** See [Pre-Release Planning - CI/CD and Build Health](#12-phase-1-pre-release-planning)

### 2.4 Documentation Readiness Gate

**Decision Point:** Is documentation ready to support the release?

**Gate Criteria:**

- ✓ Release notes draft exists with major features listed
- ✓ Known issues and limitations documented
- ✓ Compatibility matrix documented showing which component versions work together
- ✓ Breaking changes (if any) documented with migration guides and version requirements
- ✓ Upgrade/migration documentation drafted (if applicable)
- ✓ Component documentation up-to-date (hyperfleet-api, hyperfleet-sentinel, hyperfleet-adapter)

**If criteria not met:**

- **Release notes missing:** Documentation can be finalized during Code Freeze, but draft must exist
- **Breaking changes undocumented:** Block release until migration guides completed
- **Component docs outdated:** Update during stabilization phase
- **Upgrade guide missing:** Block GA release until completed (mandatory for releases with breaking changes)

**Documentation refinement:**

- Draft documentation acceptable at Feature Freeze
- Final polish and validation during Code Freeze phase
- See [Documentation Completeness](#43-documentation-completeness-mandatory) for final GA requirements

**Detailed execution checklist:** See [Pre-Release Planning - Documentation Readiness](#12-phase-1-pre-release-planning) and [Code Freeze - Documentation Finalization](#15-phase-4-code-freeze)

### 2.5 Organizational Readiness Gate

**Decision Point:** Are people and processes ready to execute the release?

**Gate Criteria:**

- ✓ Release Owner identified and assigned
- ✓ Stakeholder communication plan is in place

**If criteria not met:**

- **No Release Owner:** Assign Release Owner before proceeding (mandatory role)
- **Communication plan missing:** Define communication channels and stakeholder notification plan

**Detailed execution checklist:** See [Pre-Release Planning - Setup](#12-phase-1-pre-release-planning) and [Stakeholder Communication](#12-phase-1-pre-release-planning)

---

**Final Decision Point:** When all gates above (2.1-2.5) are met, the Release Owner can call for Feature Freeze and transition to code stabilization phase.

---

## 3. Code Freeze and Branching Strategy

### 3.1 Branching Model

HyperFleet follows a **release branch workflow** based on Kubernetes and OpenShift best practices:

```text
main (development branch)
  │
  │ Active Development Phase
  │
  ├─── release-X.Y (branch created at Feature Freeze)
  │      │
  │      │ Stabilization Phase (bug fixes only)
  │      │
  │      │ Code Freeze (critical fixes only)
  │      │
  │      ├─── vX.Y.0-rc1 (tag - Release Candidate 1)
  │      ├─── vX.Y.0-rc2 (tag - Release Candidate 2, if needed)
  │      └─── vX.Y.0 (tag - GA Release)
  │
  │ (main branch continues with X.Y+1 development)
  │
  └─── (next release cycle)

After GA:
release-X.Y (branch maintained post-release)
  │
  ├─── vX.Y.1 (tag - Z-stream/patch release, NO RC)
  ├─── vX.Y.2 (tag - Z-stream/patch release, NO RC)
  └─── ... (support window: 6 months)
```

**Note on Z-Stream (Patch) Releases:**

- Z-stream releases (vX.Y.1, vX.Y.2, etc.) **do not require Release Candidate (RC) tags**
- These releases go directly from testing to GA tag
- Scope limited to bug fixes, security patches, and minor improvements (no new features or breaking changes)
- Faster turnaround for low-risk fixes while maintaining quality gates

### 3.2 Timeline and Freeze Process

**Release Cycle:** TBD by Tech Lead after initial releases

**Timeline Breakdown:**

- **Weeks 1-2 (TBD by Tech Lead - Development Phase):** Active feature development, normal PR process on `main` branch
  - Continuous testing on all PRs (unit, integration, linting)
  - Documentation written alongside code
  - Features merged as they're completed
  - **End of Week 2 (TBD by Tech Lead):** Release Readiness Review (go/no-go decision)
- **Week 3 (TBD by Tech Lead, estimated 5 working days - Stabilization & Release Phase):**
  - **Day 1:** Feature Freeze - create `release-X.Y` branch, cut rc1
    - `main` branch reopens immediately for next release development
  - **Days 1-3:** Stabilization - bug fixes merged to main first, then cherry-picked to release branch, full E2E test suite execution (automated + manual)
  - **Days 4-5:** Code Freeze - only critical/blocker fixes with Release Owner approval, documentation finalization, final validation and sign-off
  - **End of Day 5:** GA release tagged and published with all artifacts

**Note:** Timeline can be adjusted based on actual circumstances. As the product matures and automation capabilities improve, the team should continuously refine and optimize the release cadence based on actual data and lessons learned from each release cycle.

### 3.3 Code Freeze Mechanics

**Feature Freeze:**

```bash
# Release Owner creates release branch from main (per component repo)
git checkout main
git pull origin main
git checkout -b release-1.5
git push origin release-1.5

# Create first release candidate (per component version)
git tag -a v1.5.0-rc1 -m "hyperfleet-api RC1 for v1.5.0"
git push origin v1.5.0-rc1
```

**After Feature Freeze:**

- `main` branch reopens immediately for next release (v1.6) development
- All changes to `release-1.5` branch follow the fix workflow below
- Release Owner reviews and approves all PRs to release branch

**Code Freeze:**

- Only Critical or above bug fixes allowed into release branch (Blocker, Critical)
- Each fix requires:
  - Bug severity: Critical or above (Blocker, Critical)
  - Release Owner approval
  - Successful test run in Prow
  - Risk assessment documented

**Bug Fix Workflow:**

<!-- markdownlint-disable-next-line MD036 -->
**All fixes follow the same order: Main first → Cherry-pick to release branch**

```text
Bug found during release testing
    │
    ▼
1. Fix on main first
   - Create PR to main with tests, link bug ticket
   - Merge after code review
    │
    ▼
2. Cherry-pick to release branch
   - git cherry-pick <fix-sha> onto release-X.Y
   - Create PR to release-X.Y, link SAME bug ticket
   - Release Owner approves
    │
    ▼
3. Verification (MANDATORY):
   - Original developer: Confirms cherry-pick PR merged to release branch
   - Release Owner: Verifies cherry-pick completion
   - Bug ticket CANNOT be closed until BOTH main AND release are fixed
```

```bash
# 1. Fix on main first
git checkout main
# Create PR with fix + tests, link bug ticket, merge after review

# 2. Cherry-pick to release branch
git checkout release-X.Y
git cherry-pick <fix-sha>
# If conflicts → resolve manually, create PR to release-X.Y
# If clean → create PR to release-X.Y for Release Owner approval

# 3. Link SAME bug ticket in both PRs
```

**If cherry-pick fails due to divergence:** Create the fix directly on the release branch as a separate PR. Ensure the same fix is also applied to main (it should already be there from step 1, but verify the approaches are equivalent). Document in the PR why the cherry-pick didn't apply cleanly.

### 3.4 Release Branch Maintenance

**Support Policy:** 6 months with lifecycle stages

Every release receives 6 months of support from its GA date, divided into two phases:

#### Phase 1: Full Support (first 3 months)

- All bug fixes and enhancements
- Patch releases for any severity (Major+)
- Active maintenance

#### Phase 2: Security Maintenance (months 3-6)

- CRITICAL and HIGH security vulnerabilities only
- Blocker bugs affecting production stability
- Limited patch releases

#### After 6 months: End of Life (EOL)

- No further updates
- Users must upgrade to supported version

#### Backport Decision Tree

- Version < 3 months old: Backport all Major+ severity bugs
- Version 3-6 months old: Backport only CRITICAL/HIGH CVEs and Blockers
- Version > 6 months old: EOL, no backports

### 3.5 Versioning Strategy

**HyperFleet uses independent component versioning** with validated release combinations.

Each core component (hyperfleet-api, hyperfleet-sentinel, hyperfleet-adapter) maintains its own semantic version number, evolving at its own pace based on changes and feature development.

```text
HyperFleet Release 1.5 (validated combination):
├─ hyperfleet-api: v1.5.0
├─ hyperfleet-sentinel: v1.4.2
└─ hyperfleet-adapter: v2.0.0
```

**Rationale:**

- **Flexibility:** Components can evolve independently without forcing artificial version bumps
- **Semantic accuracy:** Version numbers reflect actual changes (v1.4.2 → v1.4.3 for bug fix, not v1.4.2 → v1.5.0)
- **Efficient releases:** Components without changes don't require new versions or rebuilds
- **Clear change tracking:** Each component's version history accurately reflects its evolution
- **Industry standard:** Aligns with microservices best practices (Kubernetes components, cloud services)

**HyperFleet Release Number:**

- Defines a validated, compatibility-tested set of component versions
- Format: "HyperFleet Release X.Y" (e.g., Release 1.5, Release 1.6)
- Documented in release notes with full compatibility matrix
- Simplifies user experience: "Install HyperFleet Release 1.5" with clear component version mapping

#### 3.5.1 Branching and Tagging Rules

**Key Principles:**

1. **Independent branching:** Each component creates release branches based on its own version (e.g., `release-1.5`, `release-2.0`)
2. **Independent tagging:** Each component tags releases according to its semantic version (e.g., `v1.5.0`, `v1.4.2`, `v2.0.0`)
3. **Selective releases:** Only components with changes create new release branches and tags

**Why?** Components evolve at their own pace, and version numbers should accurately reflect actual changes.

**Component-Specific Releases:**

Between HyperFleet releases, individual components can issue releases independently:

```bash
# Example: Critical bug found in hyperfleet-sentinel after HyperFleet Release 1.5 GA
# Current: HyperFleet Release 1.5 (API v1.5.0, hyperfleet-sentinel v1.4.2, Adapter v2.0.0)

# hyperfleet-sentinel creates patch release v1.4.3
cd openshift-hyperfleet/hyperfleet-sentinel
git checkout release-1.4
# Apply fix, test
git tag -a v1.4.3 -m "hyperfleet-sentinel v1.4.3 - Hotfix for metrics bug"
git push origin v1.4.3

# Result: Users can upgrade just hyperfleet-sentinel v1.4.2 → v1.4.3 without full release
# No new HyperFleet Release number needed for single-component release
```

**When to Create Component Release vs HyperFleet Release:**

**Component Release:**

Create a component release only for isolated, fully backward-compatible fixes within a single component that do not impact APIs, schemas, cross-component behavior, or coordinated platform upgrades.

**HyperFleet Release:**

Create a HyperFleet release whenever a change affects supported platform users, introduces cross-component compatibility or contract implications, delivers security or critical stability fixes, or requires coordinated, platform-wide, validated upgrades.

**Supporting Repository Branching for HyperFleet Releases:**

When creating a HyperFleet release, the following supporting repositories also participate in the release process:

**For Major/Minor Releases (e.g., HyperFleet Release 1.5):**

1. **hyperfleet-e2e** - E2E test suites
   - Create `release-1.5` branch at Feature Freeze
   - Contains E2E tests validating the specific component combinations for this release

2. **hyperfleet-infra** - Infrastructure configurations
   - Create `release-1.5` branch at Feature Freeze (if infrastructure changes needed)
   - Contains deployment scripts, cluster configs aligned with this release

3. **hyperfleet-release** - Release coordination and documentation
   - Create `release-1.5` branch at Feature Freeze
   - Contains release notes, compatibility matrices, installation guides
   - Tag the release: `release-1.5` at GA

**For Patch Releases (e.g., HyperFleet Release 1.5.1, 1.5.2):**

Supporting repositories **do not create new branches** for patch releases:

1. **hyperfleet-e2e, hyperfleet-infra**
   - Stay on existing `release-1.5` branch
   - Commit updates/fixes to the same branch as needed

2. **hyperfleet-release**
   - Stay on existing `release-1.5` branch
   - Update release notes and compatibility matrix
   - **Create new tag** for each patch: `release-1.5.1`, `release-1.5.2`

   ```bash
   # Example: HyperFleet Release 1.5.1 (patch)
   cd openshift-hyperfleet/hyperfleet-release
   git checkout release-1.5

   # Update release notes with patch changes
   # (e.g., hyperfleet-api v1.5.0 → v1.5.1)

   # Tag the patch release
   git tag -a release-1.5.1 -m "HyperFleet Release 1.5.1 (API v1.5.1, hyperfleet-sentinel v1.4.2, Adapter v2.0.0)"
   git push origin release-1.5.1
   ```

**Rationale:** Patch releases are incremental updates on the same base release. Supporting repos use the same branch infrastructure with updated content and new tags to mark each patch version.

#### 3.5.2 Practical Example: HyperFleet Release 1.5

**Scenario:**

- hyperfleet-api: **Major feature** (new GitOps integration) → Version bump to v1.5.0
- hyperfleet-sentinel: **Bug fixes only** → Patch version bump to v1.4.2
- hyperfleet-adapter: **Breaking changes** → Major version bump to v2.0.0

**At Feature Freeze - Create Component-Specific Release Branches:**

```bash
# hyperfleet-api - MINOR version bump (new features)
cd openshift-hyperfleet/hyperfleet-api
git checkout -b release-1.5 && git push origin release-1.5

# hyperfleet-sentinel - PATCH version bump (bug fixes only)
cd openshift-hyperfleet/hyperfleet-sentinel
git checkout -b release-1.4 && git push origin release-1.4
# (or use existing release-1.4 branch if it already exists)

# hyperfleet-adapter - MAJOR version bump (breaking changes)
cd openshift-hyperfleet/hyperfleet-adapter
git checkout -b release-2.0 && git push origin release-2.0
```

**At GA Release - Tag Each Component with Its Own Version:**

```bash
# hyperfleet-api - Tag v1.5.0 (new minor version)
cd openshift-hyperfleet/hyperfleet-api
git checkout release-1.5
git tag -a v1.5.0 -m "hyperfleet-api v1.5.0 - GitOps integration"
git push origin v1.5.0

# hyperfleet-sentinel - Tag v1.4.2 (patch release)
cd openshift-hyperfleet/hyperfleet-sentinel
git checkout release-1.4
git tag -a v1.4.2 -m "hyperfleet-sentinel v1.4.2 - Memory leak fix"
git push origin v1.4.2

# hyperfleet-adapter - Tag v2.0.0 (major version)
cd openshift-hyperfleet/hyperfleet-adapter
git checkout release-2.0
git tag -a v2.0.0 -m "hyperfleet-adapter v2.0.0 - Plugin API v2"
git push origin v2.0.0
```

**Result:**

```text
HyperFleet Release 1.5 (validated combination):

Component Release Branches:
- hyperfleet-api: release-1.5
- hyperfleet-sentinel: release-1.4
- hyperfleet-adapter: release-2.0

Component Version Tags:
- hyperfleet-api: v1.5.0
- hyperfleet-sentinel: v1.4.2
- hyperfleet-adapter: v2.0.0

Container Images (for HyperFleet Release 1.5):
- quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/hyperfleet-api:1.5.0
- quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/hyperfleet-sentinel:1.4.2
- quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/hyperfleet-adapter:2.0.0

Compatibility:
- hyperfleet-api v1.5.0 requires hyperfleet-adapter ≥ v2.0.0
- hyperfleet-sentinel v1.4.2 is compatible with hyperfleet-adapter v1.x and v2.x
- Full compatibility matrix documented in release notes
```

---

## 4. Release Readiness Criteria

Before declaring a release as "GA-Ready", all the following criteria must be satisfied:

### 4.0 Build and Release Pipeline for Release Branches

Image builds from release branches are handled by Konflux via PaC tag-triggered pipelines. When an RC or release tag is pushed to a component's release branch, PaC's `.tekton/` tag pipeline triggers a Konflux build that auto-releases to Quay. No per-branch Prow build configuration is needed.

See [Konflux Release Pipeline Design](./konflux-release-pipeline-design.md) for the full pipeline architecture, CEL tag matching, and version injection.

**Prow E2E configuration for release branches:**

Prow retains E2E testing. A single reusable RC E2E job accepts image coordinates via environment variables, eliminating the need for per-release-branch job definitions. RC E2E is triggered via the `hyperfleet-release` repo tag -> GitHub Action -> Gangway -> Prow.

**Best Practices:**

- Monitor nightly E2E test results for regressions
- Disable nightly jobs when release reaches EOL (after 6 months)

### 4.1 Testing & Validation (Mandatory)

**Unit & Integration Testing:**

- ✓ All unit tests passing across all components
- ✓ Integration test suite passing

**E2E Testing:**

- ✓ Critical user workflows validated (E2E test suite)
- ✓ Backward compatibility testing with N-1 version
- ✓ Installation/upgrade path tested

**Performance & Load Testing (once performance testing is supported):**

- ✓ Performance benchmarks show no regression > 10% vs. previous release
- ✓ Load testing validates system handles expected production load
- ✓ Resource utilization (CPU, memory) within acceptable bounds

**Note:** Automated testing is preferred for all scenarios. If a test scenario is not yet automated, it must be executed manually before release approval.

### 4.2 Bug Severity Gates (Mandatory)

- ✓ No open bugs with severity **Major** or above (Blocker, Critical, Major)
- ✓ `Normal` bugs are tracked for follow-up and do not block GA unless explicitly elevated by Release Owner + Tech Lead
- ✓ Minor bugs:  No gate, tracked for future releases

### 4.3 Documentation Completeness (Mandatory)

**Release Documentation:**

- ✓ Release notes finalized (what's new, bug fixes, breaking changes)
- ✓ Known issues and limitations documented
- ✓ Deprecation notices published (if applicable)

**Operational Documentation:**

- ✓ Installation guide updated
- ✓ Upgrade instructions complete (N-1 → N)
- ✓ Deployment runbook created and reviewed

**Technical Documentation:**

- ✓ Component documentation updated (if changed):
  - hyperfleet-api documentation
  - hyperfleet-sentinel documentation
  - hyperfleet-adapter documentation
- ✓ Configuration changes documented per component

### 4.4 Cross-Team Coordination

- ⚠ Integration validation with dependent offerings (TBD)

### 4.5 Security & Compliance

**Mandatory (All Phases):**

- ✓ Vulnerability scanning: No CRITICAL/HIGH CVEs in container images

**Active (Provided by Konflux):**

- ✓ Supply chain security: SLSA Level 3 provenance generated (Tekton Chains)
- ✓ Software Bill of Materials (SBOM) generated automatically
- ✓ Container images signed with Sigstore/Cosign (keyless, OIDC-based)
- ✓ Enterprise Contract Policy enforcement (`app-interface-standard`)

See [Konflux Release Pipeline Design — Enterprise Contract Policy](./konflux-release-pipeline-design.md#7-enterprise-contract-policy) for details on what the policy validates and planned custom policy enhancements.

### 4.6 Release Artifacts Verification (Mandatory)

- ✓ All container images built and pushed to registry
- ✓ Helm charts packaged and tested
- ✓ Git tags created with correct version
- ✓ Release artifacts checksums/signatures verified

**Gate Decision:** Only when ALL mandatory criteria are met can the release be declared GA-ready and published.

---

## 5. Bug Handling Workflow After Code Freeze

### 5.1 Bug Triage Process

When a bug is discovered after code freeze (during RC testing or late in release cycle):

```text
            Bug Reported
                 │
                 ↓
      ┌─────────────────────────┐
      │ Initial Assessment      │
      │ - Severity assignment   │
      │ - Reproducibility       │
      │ - Impact analysis       │
      └───────────┬─────────────┘
                  │
                  ↓
┌────────────────────────────────────────────┐
│          Severity-Based Routing            │
├──────────────┬──────────────┬──────────────┤
│ Blocker/     │    Major     │ Normal/Minor │
│ Critical     │              │              │
└──────┬───────┴──────┬───────┴──────┬───────┘
       │              │              │
       ↓              ↓              ↓
  [FIX NOW]    [TRIAGE MEETING]  [DEFER]
```

### 5.2 Decision Framework

**For Blocker/Critical bugs:**

1. **Immediate Action:** Developer assigned within 2 hours
2. **Fix & Test:** Root cause analysis, fix implementation, automated tests added
3. **Fix Main First:** Fix applied to main, then cherry-picked to release branch (see [Bug Fix Workflow](#33-code-freeze-mechanics))
   - Create PR to release branch with bug ticket linked
   - Bug ticket stays open until main is also fixed
   - Developer and Release Owner verify both PRs merged
4. **New RC:** Cut new release candidate (e.g., v1.5.0-rc3)
5. **Regression Testing:** Full test suite re-run
6. **Time Box:** If fix takes > 1 working day, consider release delay or degrading severity

**For Major bugs:**

1. **Before Code Freeze:** Major severity bugs must be fixed before GA release
   - **Release Owner Assessment:** Evaluate impact, risk, complexity, and timeline
   - **Fix & Include:** Implement fix on main, cherry-pick to release branch (see [Bug Fix Workflow](#33-code-freeze-mechanics))
   - **If Not Fixable in Timeline:**
     - Consider release delay to allow fix completion
     - OR downgrade severity if impact assessment justifies (requires stakeholder approval and documented rationale)
2. **During Code Freeze:** Major bugs must be either:
   - **Escalated to Critical:** With documented justification showing blocker-level impact to be included in the release
   - **Deferred to Next Patch:** Scheduled for the next patch release (e.g., v1.5.1) if impact does not warrant Critical escalation

**For Normal/Minor bugs:**

- Default: Defer to next patch release or next minor release
- Track in backlog for future releases

### 5.3 Post-Code Freeze PR Approval Process

All PRs to release branch after code freeze require:

1. **Justification:** PR description must include:
   - Bug severity and impact
   - Why fix cannot wait for patch release
   - Risk assessment (What could break?)
   - Test coverage added/modified

2. **Approval Chain:**

   ```text
   Developer → Code Review → Release Owner → Automated Tests → Merge
   ```

   - Minimum 2 approvals (1 technical reviewer + Release Owner)
   - Prow tests must be green
   - No approval bypasses allowed

3. **Communication:**
   - Post to release Slack channel for visibility
   - Update release ticket
   - Notify stakeholders if fix delays GA timeline

### 5.4 Hotfix Workflow (Post-GA)

For bugs discovered after GA release:

```bash
# Example assumes hyperfleet-sentinel component (v1.4.x)

# 1. Fix on main first
git checkout main
# Create PR with fix + tests, link bug ticket, merge after review

# 2. Cherry-pick to release branch
git checkout release-1.4
git cherry-pick <fix-sha>
git push origin release-1.4

# 3. Tag component patch release (no RC needed for patches)
git tag -a v1.4.3 -m "Patch release v1.4.3"
git push origin v1.4.3
# → PaC tag.yaml triggers → Konflux builds → auto-release to Quay

# 4. Smoke test against new image, update RELEASE_MANIFEST.yaml
```

**Hotfix Release Timeline:**

- Blocker/Critical CVE: Patch release within 1 working day

Non-critical issues do not use the hotfix process — they follow the standard patch release workflow (2 working days) or are deferred to the next release.

### 5.5 Release Recovery Strategy

**HyperFleet uses a roll-forward recovery strategy for MVP releases.**

#### 5.5.1 Roll-Forward (Primary Strategy)

When issues are discovered in a GA release, the default recovery path is to **fix forward** via a patch release:

**Process:**

1. Fix the issue on main first (link bug ticket)
2. Cherry-pick fix to release branch (same bug ticket, stays open until both merged)
3. Cut patch release (e.g., v1.5.0 → v1.5.1)
4. Deploy patch following standard deployment procedures
5. Verify cherry-pick completion (both developer and Release Owner)

**Timeline:**

- Blocker/Critical CVE: Patch release within 1 working day
- All other issues: Patch release within 2 working days, or defer to next release

**Advantages:**

- Simpler testing scope (only test the fix)
- No database migration reversal complexity
- Maintains forward version progression
- Faster response for critical issues

#### 5.5.2 Rollback Support (Post-MVP)

**Status:** Deferred (separate epic required)

**Reference:** See [versioning trade-offs documentation](https://github.com/openshift-hyperfleet/architecture/blob/main/hyperfleet/docs/versioning-trade-offs.md#4-database-migration-and-rollback-procedures-post-mvp) for detailed rollback considerations.

**Decision Point:** Evaluate rollback support necessity after MVP based on:

- Incident frequency and severity
- Customer requirements for rollback capabilities
- Database schema stability
- Testing infrastructure maturity

---

## 6. Release Cadence

**Regular Releases:** TBD - to be determined by Tech Lead after initial releases

- **Weeks 1-2 (TBD by Tech Lead)**: Active development
- **End of Week 2 (TBD by Tech Lead)**: Release Readiness Review (go/no-go decision)
- **Week 3 (TBD by Tech Lead, estimated 5 working days)**:
  - Day 1: Feature freeze, create release branches
  - Days 1-3: Stabilization and testing
  - Days 4-5: Code freeze, final validation
  - End of Day 5: GA release
  - Note: Timeline can be adjusted based on actual circumstances

**Ad-Hoc Releases:** As needed for urgent requirements (3-5 working days)

- Triggers: Critical bugs, security vulnerabilities, urgent business needs
- Reduced testing scope (unit, integration, E2E for affected components only)
- Release Owner approval required

**Cadence Refinement:**

- Revisit cadence after every several releases during retrospectives
- Adjust based on team capacity, automation maturity, and quality metrics

---

## 7. Release Artifacts and Deliverables

### 7.0 Release Repository

**Create: `openshift-hyperfleet/hyperfleet-release` repository**

A dedicated release repository serves as the single source of truth for all HyperFleet releases.

**Purpose:**

- Centralized release notes, installation guides, and upgrade documentation
- Defines validated component version combinations for each HyperFleet Release
- Release tracking issues and automation scripts
- User-facing documentation via GitHub Pages (optional)

**What goes in the release repository:**

- HyperFleet Release tags: `release-1.5`, `release-1.6` (marking validated component combinations)
- Release notes for each HyperFleet Release (`releases/release-1.5/release-notes.md`)
- Compatibility matrix for each release (which component versions work together)
- Installation and upgrade guides
- Links to component container images and artifacts
- Aggregated changelog from all components
- Release automation scripts

**What stays in component repositories:**

- Source code
- Component-specific git tags (e.g., `v1.5.0`, `v1.4.2`, `v2.0.0`)
- Component-specific Helm charts
- Component-specific CHANGELOGs
- Development documentation

**Benefits:** Single location for offering team to find all release information, clear component version combinations for each release, cleaner separation between component development and integrated release artifacts.

### 7.1 Primary Release Artifacts

**Container Images:**

- Built automatically by Konflux on release tag push (PaC tag-triggered pipeline)
- Published to `quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/`
- Includes signed provenance (SLSA Level 3) and SBOM
- Registered in Pyxis for Red Hat vulnerability scanning
- **Image naming:** Each component uses its own independent semantic version

  ```text
  quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/hyperfleet-{component}:{component-version}
  ```

  - `hyperfleet-api` for hyperfleet-api
  - `hyperfleet-sentinel` for hyperfleet-sentinel
  - `hyperfleet-adapter` for hyperfleet-adapter
- **Example for HyperFleet Release 1.5:**
  - `quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/hyperfleet-api:1.5.0`
  - `quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/hyperfleet-sentinel:1.4.2`
  - `quay.io/redhat-services-prod/hyperfleet-tenant/hyperfleet/hyperfleet-adapter:2.0.0`

  (Each component has its own version reflecting its actual changes per [Versioning Strategy](#35-versioning-strategy))

**Helm Charts:**

- Each component has its own Helm chart in component repositories
  - Note: hyperfleet-adapter base chart is designed for overlay usage by business adapters, not standalone deployment
- Component charts are published as OCI artifacts to Quay.io via Konflux — see [Helm OCI Distribution Design](./helm-oci-distribution-design.md)
- `hyperfleet-infra` repo contains deployment umbrella charts for local dev and E2E testing (not published to OCI)

**Git Tags:**

- Git tags created independently in component repositories: `vX.Y.Z` (reflecting each component's version)
- HyperFleet Release tag created in `openshift-hyperfleet/hyperfleet-release` repository: `release-X.Y` (single source of truth for the validated component combination - see [Release Repository](#70-release-repository))
- GitHub Releases created from tags with release notes and compatibility matrix

### 7.2 Documentation Deliverables

#### 7.2.1 Release Notes

**Purpose:** Communicate what's new, breaking changes, known issues, and compatibility information to users.

**Required Sections Template:**

```markdown
# HyperFleet Release X.Y

## Overview
Brief description of release theme and major highlights

## What's New
### New Features
- **component-name vX.Y.Z:** Feature description

### Enhancements
- **component-name vX.Y.Z:** Enhancement description

## Breaking Changes
- **component-name vX.Y.Z:** Breaking change description (migration guide: link)

## Bug Fixes
- **component-name vX.Y.Z:** Bug fix description (#issue-number)

## Known Issues
- **component-name:** Issue description (workaround: description)

## Upgrade Instructions
See [Upgrade Guide](docs/upgrade-to-release-X.Y.md) for detailed instructions.

## Compatibility Matrix

**HyperFleet Release X.Y** (validated component combination)

| Component | Version | Changes | Notes |
|-----------|---------|---------|-------|
| hyperfleet-api | **vX.Y.Z** | MINOR/MAJOR/PATCH | Summary of changes |
| hyperfleet-sentinel | **vX.Y.Z** | MINOR/MAJOR/PATCH | Summary of changes |
| hyperfleet-adapter | **vX.Y.Z** | MINOR/MAJOR/PATCH | Summary of changes |

**Component Compatibility:**
- List specific version dependencies between components

**Platform Compatibility:**

| Platform | Supported Versions |
|----------|-------------------|
| Kubernetes | X.YY - X.YY |
| Helm | X.YY+ |

## Security
- **component-name vX.Y.Z:** Security-related changes, CVE fixes
```

**Execution checklist:** See [Code Freeze - Documentation Finalization](#15-phase-4-code-freeze)

#### 7.2.2 Upgrade/Installation Guide

**Purpose:** Enable users to install fresh or upgrade from N-1 version successfully.

**Required Content:**

- **Prerequisites:** Kubernetes version requirements, cluster permissions, dependencies
- **Fresh Installation Steps:** Complete installation procedure from scratch
- **Upgrade Path (N-1 → N):** Step-by-step upgrade instructions with version-specific considerations
- **Component-Specific Notes:**
  - hyperfleet-api: Installation and configuration
  - hyperfleet-sentinel: Deployment and monitoring setup
  - hyperfleet-adapter: Deployment via business adapter overlay (not standalone deployment)
- **Post-Installation Validation:** Steps to verify successful installation/upgrade
- **Troubleshooting:** Common issues and solutions
- **Rollback Procedure (if applicable):** Recovery steps if upgrade fails

**Execution checklist:** See [Code Freeze - Documentation Finalization](#15-phase-4-code-freeze)

#### 7.2.3 Component Documentation

**Purpose:** Provide technical reference and usage documentation for each component.

**Required Content by Component:**

**hyperfleet-api:**

- OpenAPI/Swagger specification (auto-generated from code)
- API reference documentation (published via Swagger UI or similar)
- Code examples demonstrating new endpoints and features
- Deprecation notices for old APIs with migration timeline
- Authentication and authorization documentation

**hyperfleet-sentinel:**

- Deployment documentation (installation, configuration)
- Monitoring and metrics documentation (dashboards, alerts)
- Configuration reference (all configuration options explained)
- Troubleshooting guide (common issues and solutions)
- Performance tuning guidelines

**hyperfleet-adapter:**

- Operator guide (how to use and extend the adapter)
- Code examples for custom adapter development
- Migration guides for breaking changes (with version-specific instructions)
- Plugin API reference
- Best practices for adapter development

**Quality Requirements:**

- All code examples must be tested and validated
- Breaking changes must be clearly highlighted
- Migration paths documented for deprecated features
- Version compatibility clearly stated

**Execution checklist:** See [Code Freeze - Documentation Finalization](#15-phase-4-code-freeze)

#### 7.2.4 Change Log

**Two-Level Changelog Approach:**

1. **Component-level:** Changelog content included in git tag messages when creating component tags
2. **HyperFleet Release-level:** Complete CHANGELOG.md in `hyperfleet-release` repo aggregating all component changes

**Component Tag Message Format:**

When creating component tags, include a structured changelog in the tag message following the Keep a Changelog standard:

**Example - hyperfleet-api v1.5.0 tag message:**

```markdown
## hyperfleet-api v1.5.0 - GitOps integration

### Added
- New GitOps integration for ROSA deployments
- OAuth 2.1 authentication support

### Changed
- Updated authentication flow to use OAuth 2.1
- Improved API response caching mechanism

### Removed
- `/v1/legacy-auth` endpoint (deprecated since v1.3.0)

### Fixed
- Authentication token refresh issue (#198)
- Race condition in concurrent API requests (#210)

### Security
- Updated dependencies to address CVE-2026-1234
```

**HyperFleet Release CHANGELOG (hyperfleet-release repo):**

The `hyperfleet-release` repository contains a complete CHANGELOG.md that aggregates changes from all components for each HyperFleet release:

```markdown
# HyperFleet Release Changelog

## Release 1.5 - 2026-05-12

### Component Versions
- hyperfleet-api: v1.5.0
- hyperfleet-sentinel: v1.4.2
- hyperfleet-adapter: v2.0.0

### hyperfleet-api v1.5.0

#### Added
- New GitOps integration for ROSA deployments
- OAuth 2.1 authentication support

#### Changed
- Updated authentication flow to use OAuth 2.1
- Improved API response caching mechanism

#### Removed
- `/v1/legacy-auth` endpoint (deprecated since v1.3.0)

#### Fixed
- Authentication token refresh issue (#198)
- Race condition in concurrent API requests (#210)

### hyperfleet-sentinel v1.4.2

#### Fixed
- Memory leak in metrics collector (#234)
- Race condition in concurrent monitoring (#256)

### hyperfleet-adapter v2.0.0

#### Added
- Plugin API v2 with enhanced extensibility
- Support for async plugin initialization

#### Changed
- **BREAKING:** Plugin API v2 replaces v1 (migration guide: docs/plugin-migration-v2.md)
- Improved plugin loading performance by 40%

#### Removed
- **BREAKING:** Plugin API v1 support

### Security Updates (All Components)
- Updated dependencies to address CVE-2026-1234, CVE-2026-5678
```

**Standards:**

- Component tag messages must include structured changelog content following Keep a Changelog format
- HyperFleet Release CHANGELOG.md aggregates all component changes
- All changes categorized: Added/Changed/Removed/Fixed/Security
- Include links to PRs/issues for traceability
- Security fixes and breaking changes clearly highlighted

**Execution checklist:** See [GA Release - Tag GA Release](#16-phase-5-ga-release)

### 7.3 Compliance and Security Artifacts

**Active (Provided by Konflux):**

- Vulnerability scanning of container images
- No CRITICAL/HIGH CVEs in release
- Enterprise Contract Policy enforcement (`app-interface-standard`)
- SBOM generation
- SLSA Level 3 provenance
- Image signing with Sigstore/Cosign

---

## 8. Konflux vs. Prow Comparison

### 8.1 Current State: Prow

**What Works:**

- Automated CI/CD pipeline (testing, image builds)
- GitHub integration
- Team familiarity

**What's Missing:**

- SLSA provenance, SBOM generation, image signing
- Requires additional tooling for supply chain security

### 8.2 Konflux Benefits

**Supply Chain Security:**

- SLSA Level 3 provenance, SBOM, Sigstore signing (built-in)
- **Enterprise Contract Policy enforcement**
- Integrated vulnerability scanning

**Release Automation:**

- Unified build-test-release workflow
- OCI artifact management
- Policy-as-code compliance gates

### 8.3 Current Approach

**Hybrid Model (Active):**

- **Konflux** handles container image build, signing, and release to Quay
- **Prow** retains PR presubmit checks and E2E testing (nightly + RC)
- Supply chain security (SLSA L3, SBOM, EC) provided by Konflux
- See [Konflux Release Pipeline Design](./konflux-release-pipeline-design.md) for full architecture

---

## 9. Next Steps

### 9.1 Post-First-Release Improvements

#### 9.1.1 Conduct Retrospectives and Identify Improvements

After completing the first few releases with manual processes, conduct retrospectives to:

- Identify workflow pain points and bottlenecks
- Determine which manual steps should be automated
- Evaluate release process effectiveness (timing, quality gates, coordination)
- Gather feedback from Release Owners, developers, and stakeholders
- Update release procedures based on lessons learned
- Prioritize automation opportunities (Helm packaging, release notes generation, GitHub Releases)

#### 9.1.2 Konflux Onboarding (In Progress)

Konflux onboarding is tracked under [HYPERFLEET-830](https://redhat.atlassian.net/browse/HYPERFLEET-830). The pipeline design is documented in [Konflux Release Pipeline Design](./konflux-release-pipeline-design.md) and [ADR 0014](../../adrs/0014-konflux-build-and-release.md).

**Remaining steps:**

- Create `hyperfleet` tenant workspace on kflux-prd-rh02
- Create RPA and constraint files in `konflux-release-data`
- Add `.tekton/` pipeline definitions to each component repo
- Update nightly Prow E2E to pull from Quay
- Configure RC E2E via Gangway
- Remove legacy Prow image build configs

#### 9.1.3 Additional Process Improvements

Based on retrospective findings and Konflux capabilities:

- Establish automated E2E test gate as mandatory release criteria
- Create release health monitoring dashboards
- Define SLI/SLO framework for release quality metrics
- Optimize release cadence based on data (6-month review)
- Consider LTS release designation (e.g., every 4th release)

### 9.2 Success Metrics

**Track and review quarterly:**

**Release Metrics:**

- HyperFleet Release frequency (cadence TBD by Tech Lead based on team capacity and automation maturity)
- Component patch release frequency (individual component updates between HyperFleet Releases)
- Code freeze duration (target: < 1 week, ideally 3-5 days)
- On-time delivery (target: > 80% of HyperFleet Releases on schedule)
- Stabilization phase variance (track actual vs. planned to inform process improvements)

**Quality Metrics:**

- Bug escape rate (bugs found post-GA per component and per HyperFleet Release)
- Hotfix frequency per component (target: < 2 patch releases per component between HyperFleet Releases)
- Mean time to patch critical vulnerabilities (target: < 2 working days for any component)
- Cross-component compatibility issues found in production (target: 0)

---

## 10. Appendices

### Appendix A: References and Sources

**Kubernetes Release Process:**

- [Kubernetes Release Cycle](https://kubernetes.io/releases/release/)
- [Kubernetes Release Cadence](https://goteleport.com/blog/kubernetes-release-cycle/)
- [Patch Releases | Kubernetes](https://kubernetes.io/releases/patch-releases/)
- [Kubernetes Branch](https://github.com/kubernetes/kubernetes/branches)

**Konflux CI/CD:**

- [Why Konflux?](https://konflux-ci.dev/docs/)
- [How we use software provenance at Red Hat](https://developers.redhat.com/articles/2025/05/15/how-we-use-software-provenance-red-hat)
- [Konflux Release Data Flow](https://konflux.pages.redhat.com/docs/users/releasing/preparing-for-release.html)

**Release Artifacts:**

- [OCI Artifacts Explained: Beyond Container Images](https://oneuptime.com/blog/post/2025-12-08-oci-artifacts-explained/view)
- [Manage Helm charts | Artifact Registry](https://docs.cloud.google.com/artifact-registry/docs/helm/manage-charts)

**Bug Handling and Code Freeze:**

- [Mastering the Code Freeze Process](https://ones.com/blog/mastering-code-freeze-process-software-stability/)
- [Code Freezes and Feature Flags](https://devcycle.com/blog/code-freezes-and-feature-flags)

**Cloud Readiness:**

- [The Ultimate Cloud Readiness Checklist for 2026](https://www.pulsion.co.uk/blog/cloud-readiness-checklist/)
- [Production Readiness Checklist](https://gruntwork.io/devops-checklist/)

### Appendix B: Glossary

See [Release Glossary](./glossary.md).

### Appendix C: Template - Ad-Hoc Release Request

```markdown
# Ad-Hoc Release Request: HyperFleet Release 1.5.1 (or Component-Specific Patch)

## Release Type
- [ ] **Full HyperFleet Release** (multiple components, validated combination)
- [ ] **Single Component Patch** (e.g., hyperfleet-sentinel v1.4.3 only, no new HyperFleet Release number)

## Requestor Information
- **Requested by:** @username
- **Request date:** YYYY-MM-DD
- **Urgency:** Critical / High / Medium
- **Target release date:** YYYY-MM-DD

## Justification
Why can't this wait for the next regular release (HyperFleet Release X.X on DATE)?

[Explain business justification, customer impact, or urgency]

## Scope
What will be included in this ad-hoc release?

### Component Version Changes

| Component | Current Version (Release 1.5) | New Version | Change Type | Reason |
|-----------|------------------------------|-------------|-------------|--------|
| hyperfleet-api | v1.5.0 | v1.5.1 | PATCH | Critical security fix |
| hyperfleet-sentinel | v1.4.2 | v1.4.2 | No change | - |
| hyperfleet-adapter | v2.0.0 | v2.0.0 | No change | - |

### Changes Included
- [ ] Feature/fix #1: Brief description
- [ ] Feature/fix #2: Brief description
- [ ] Bug fix #3: Brief description

### Changes Explicitly Excluded
List what is NOT included to keep scope tight:
- Other pending PRs
- Unrelated bug fixes
- Nice-to-have features

## Impact Assessment

### Components Affected
- [ ] hyperfleet-api - [changes description]
- [ ] hyperfleet-sentinel - [changes description]
- [ ] hyperfleet-adapter - [changes description]

### Risk Level
- [ ] Low - Minor change, well-tested, quick patch if needed
- [ ] Medium - Moderate change, some risk
- [ ] High - Significant change, complex fix-forward required

### Blast Radius
- Number of users/environments affected: [estimate]
- Customer impact if issue occurs: [description]

## Testing Plan

### Automated Testing (Mandatory)
- [ ] Unit tests added/updated
- [ ] Integration tests passing
- [ ] E2E tests for affected components passing
- [ ] CI pipeline green

### Manual Testing (Mandatory)
- [ ] Smoke test plan defined
- [ ] Critical user paths validated
- [ ] Regression testing for affected areas

### Testing Deferred
What testing will be deferred to next regular release?
- [ ] Full exploratory testing
- [ ] Performance regression testing
- [ ] Other: [specify]

## Recovery Plan
**Primary strategy: Roll-forward via patch release (MVP approach)**

### If Issues Discovered
- [ ] Hotfix patch release plan documented
- [ ] Fix timeline estimated (target: < 2 working days for critical)
- [ ] Workaround available for users (if applicable)

### Database Migration Considerations
- [ ] Schema changes included? [yes/no]

## Stakeholder Coordination

### Offering Team Notification
- [ ] Offering team notified (minimum 2 working days advance)
- [ ] Integration testing with GCP completed
- [ ] Deployment coordination confirmed

### Communication Plan
- [ ] Release notes drafted
- [ ] Stakeholders notified
- [ ] Customer communication prepared (if external)

## Release Owner Approval

**Decision:**
- [ ] **Approved** - Proceed with ad-hoc release
- [ ] **Rejected** - Defer to next regular release (DATE)
- [ ] **Needs More Info** - [specify what's needed]

**Approval by:** @Technical Lead and @Manager
**Date:** YYYY-MM-DD
**Conditions:** [any special conditions or requirements]

## Release Timeline

**Target: 3-5 working days**

| Day | Date | Activities | Owner |
|-----|------|-----------|-------|
| 1 | YYYY-MM-DD | Development + unit tests | @dev-team |
| 2 | YYYY-MM-DD | Code review + CI tests | @reviewers |
| 3 | YYYY-MM-DD | E2E testing + RC build | @qa-team |
| 4 | YYYY-MM-DD | Manual testing + stakeholder review | @qa + @stakeholders |
| 5 | YYYY-MM-DD | GA release + deployment | @release-owner |

## Post-Release Monitoring
- [ ] Metrics dashboard monitored for 1 working day
- [ ] No error rate increase
- [ ] No performance degradation
- [ ] Post-release review completed
```

---
