# HyperFleet Release Process - Spike Report

**Document Status:** Draft
**Date:** 2026-01-29
**Last Updated:** 2026-02-10

---

## Executive Summary

This spike report defines a comprehensive release process for HyperFleet (API service, Sentinel, and Adapter Framework). The proposed process balances agility with stability, leveraging existing Prow infrastructure while establishing clear gates, workflows, and artifacts for production releases.

**Key Recommendations:**
- **Hybrid release cadence:** Regular 3-week (1-sprint) releases for quality + ad-hoc releases for urgent requirements
- Git branching strategy with release branches and cherry-pick workflow
- **Independent component versioning:** Each component (API Service, Sentinel, Adapter Framework) maintains its own semantic version
- Validated HyperFleet releases defined by compatibility-tested component version combinations
- Multi-gate release readiness criteria including automated and manual validation
- Structured bug triage workflow post-code freeze
- Comprehensive release artifacts including container images, Helm charts, and documentation
- **Start with Prow + manual release for MVP**, plan Konflux migration (with Enterprise Contract Policy support) for Post-MVP

---

## 1. Release Entry Criteria

These criteria determine when the development team can initiate the release process and enter code freeze.

### 1.1 Feature Completeness Criteria

**Feature Freeze Gate:**
- ✓ All planned features for the release milestone are code-complete
- ✓ Feature flags are in place for any experimental or incomplete features
- ✓ Feature documentation is drafted (can be finalized during code freeze)

**Technical Debt Assessment:**
- ✓ No known security vulnerabilities rated HIGH or CRITICAL remain unaddressed
- ✓ Technical debt has been reviewed and acceptable items are explicitly deferred to next release
- ✓ All deprecated APIs have migration paths documented

### 1.2 Testing & Quality Gates

- **CI/CD Pipeline Health:** Prow CI pipeline is green for all components on the main branch, validating:
  - Unit tests: ≥70% coverage for new code
  - Integration tests: Passing consistently
  - E2E tests: Critical user journeys validated
  - Performance regression tests: No degradation >10% vs. previous release

- **Build**:
  - Container images: Build successfully for all target architectures
  - Helm charts: Package without errors

### 1.3 Cross-Component Dependencies

**Version Compatibility:**
- ✓ Each component uses independent semantic versioning (see Section 2.5)
- ✓ HyperFleet Release X.Y defines a validated, compatibility-tested set of component versions
- ✓ Compatibility matrix documented showing which component versions work together
- ✓ Breaking changes (if any) are documented with migration guides and version requirements
- ✓ Backward compatibility: Each component supports N-1 version upgrade paths independently
- ✓ Cross-component API contracts validated during integration testing

### 1.4 Documentation Readiness

- ✓ Release notes draft exists with major features listed
- ✓ Known issues and limitations are documented
- ✓ Upgrade/migration documentation is drafted (if applicable)
- ✓ API documentation is up-to-date

### 1.5 Organizational Readiness

- ✓ Release Owner identified and assigned
- ✓ Stakeholder communication plan is in place

**Decision Point:** When all criteria above are met, the Release Owner can call for Feature Freeze and transition to code stabilization phase.

---

## 2. Code Freeze and Branching Strategy

### 2.1 Branching Model

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
  │      ├─── vX.Y.0-rc.1 (tag - Release Candidate 1)
  │      ├─── vX.Y.0-rc.2 (tag - Release Candidate 2, if needed)
  │      └─── vX.Y.0 (tag - GA Release)
  │
  │ (main branch continues with X.Y+1 development)
  │
  └─── (next release cycle)

After GA:
release-X.Y (branch maintained post-release)
  │
  ├─── vX.Y.1 (tag - Patch release via cherry-picks)
  ├─── vX.Y.2 (tag - Patch release)
  └─── ... (support window: 6 months)
```

### 2.2 Timeline and Freeze Process

**Sprint-Based Release Cycle (3 weeks / 1 sprint):**

**Timeline Breakdown:**
- **Weeks 1-2 (Development Phase):** Active feature development, normal PR process on `main` branch
  - Continuous testing on all PRs (unit, integration, linting)
  - Documentation written alongside code
  - Features merged as they're completed
- **Week 3 (Stabilization & Release Phase):**
  - Feature Freeze - create `release-X.Y` branch, cut RC.1
  - `main` branch reopens immediately for next release development
  - Bug fixes cherry-picked from `main`
  - Full E2E test suite execution (automated + manual)
  - Documentation finalization
  - Code Freeze - only critical/blocker fixes with Release Owner approval
  - Final validation and sign-off
  - GA release tagged and published with all artifacts

**Note:** This 3-week cadence is designed for HyperFleet as a new product requiring rapid feedback cycles with pillar teams. Since CI/CD automation is currently being built, the Stabilization & Release Phase (Week 3) may take longer in the initial releases. As the product matures and automation capabilities improve, the team should continuously refine and optimize the release cadence based on actual data and lessons learned from each release cycle.

### 2.3 Code Freeze Mechanics

**Feature Freeze:**
```bash
# Release Owner creates release branch from main (per component repo)
git checkout main
git pull origin main
git checkout -b release-1.5
git push origin release-1.5

# Create first release candidate (per component version)
git tag -a v1.5.0-rc.1 -m "API Service RC1 for v1.5.0"
git push origin v1.5.0-rc.1
```

**After Feature Freeze:**
- `main` branch reopens immediately for next release (v1.6) development
- All changes to `release-1.5` branch must be cherry-picked from `main` or created as hotfix PRs
- Release Owner reviews and approves all PRs to release branch

**Code Freeze:**
- Only Critical or above bug fixes allowed into release branch (Blocker, Critical)
- Each fix requires:
  - Bug severity: Critical or above (Blocker, Critical)
  - Release Owner approval
  - Successful test run in Prow
  - Risk assessment documented

**Cherry-Pick Process:**

**Standard Fix (Preferred):**
1. Create PR to fix bug in `main` branch → Merge
2. Cherry-pick the fix to release branch via new PR → Release Owner approves

**Release-Specific Fix (Only if bug doesn't exist in main):**
1. Create PR directly to `release-X.Y` branch → Release Owner approves

### 2.4 Release Branch Maintenance

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

#### Backport Decision Tree:
- Version < 3 months old: Backport all Major+ severity bugs
- Version 3-6 months old: Backport only CRITICAL/HIGH CVEs and Blockers
- Version > 6 months old: EOL, no backports

### 2.5 Versioning Strategy

**HyperFleet uses independent component versioning** with validated release combinations.

Each core component (API Service, Sentinel, Adapter Framework) maintains its own semantic version number, evolving at its own pace based on changes and feature development.

```text
HyperFleet Release 1.5 (validated combination):
├─ api-service: v1.5.0
├─ sentinel: v1.4.2
└─ adapter-framework: v2.0.0
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

#### 2.5.1 Branching and Tagging Rules

**Key Principles:**
1. **Independent branching:** Each component creates release branches based on its own version (e.g., `release-1.5`, `release-2.0`)
2. **Independent tagging:** Each component tags releases according to its semantic version (e.g., `v1.5.0`, `v1.4.2`, `v2.0.0`)
3. **Selective releases:** Only components with changes create new release branches and tags

**Why?** Components evolve at their own pace, and version numbers should accurately reflect actual changes.

**Component-Specific Releases:**

Between HyperFleet releases, individual components can issue releases independently:

```bash
# Example: Critical bug found in Sentinel after HyperFleet Release 1.5 GA
# Current: HyperFleet Release 1.5 (API v1.5.0, Sentinel v1.4.2, Adapter v2.0.0)

# Sentinel creates patch release v1.4.3
cd openshift-hyperfleet/sentinel
git checkout release-1.4
# Apply fix, test
git tag -a v1.4.3 -m "Sentinel v1.4.3 - Hotfix for metrics bug"
git push origin v1.4.3

# Result: Users can upgrade just Sentinel v1.4.2 → v1.4.3 without full release
# No new HyperFleet Release number needed for single-component release
```

**When to Create Component Release vs HyperFleet Release:**

**Component Release:**

Create a component release only for isolated, fully backward-compatible fixes within a single component that do not impact APIs, schemas, cross-component behavior, or coordinated platform upgrades.

**HyperFleet Release:**

Create a HyperFleet release whenever a change affects supported platform users, introduces cross-component compatibility or contract implications, delivers security or critical stability fixes, or requires coordinated, platform-wide, validated upgrades.

#### 2.5.2 Practical Example: HyperFleet Release 1.5

**Scenario:**
- API Service: **Major feature** (new GitOps integration) → Version bump to v1.5.0
- Sentinel: **Bug fixes only** → Patch version bump to v1.4.2
- Adapter Framework: **Breaking changes** → Major version bump to v2.0.0

**At Feature Freeze - Create Component-Specific Release Branches:**

```bash
# API Service - MINOR version bump (new features)
cd openshift-hyperfleet/api-service
git checkout -b release-1.5 && git push origin release-1.5

# Sentinel - PATCH version bump (bug fixes only)
cd openshift-hyperfleet/sentinel
git checkout -b release-1.4 && git push origin release-1.4
# (or cherry-pick to existing release-1.4 if it exists)

# Adapter Framework - MAJOR version bump (breaking changes)
cd openshift-hyperfleet/adapter-framework
git checkout -b release-2.0 && git push origin release-2.0
```

**At GA Release - Tag Each Component with Its Own Version:**

```bash
# API Service - Tag v1.5.0 (new minor version)
cd openshift-hyperfleet/api-service
git checkout release-1.5
git tag -a v1.5.0 -m "API Service v1.5.0 - GitOps integration"
git push origin v1.5.0

# Sentinel - Tag v1.4.2 (patch release)
cd openshift-hyperfleet/sentinel
git checkout release-1.4
git tag -a v1.4.2 -m "Sentinel v1.4.2 - Memory leak fix"
git push origin v1.4.2

# Adapter Framework - Tag v2.0.0 (major version)
cd openshift-hyperfleet/adapter-framework
git checkout release-2.0
git tag -a v2.0.0 -m "Adapter Framework v2.0.0 - Plugin API v2"
git push origin v2.0.0
```

**Result:**
```text
HyperFleet Release 1.5 (validated combination):

Component Release Branches:
- api-service: release-1.5
- sentinel: release-1.4
- adapter-framework: release-2.0

Component Version Tags:
- api-service: v1.5.0
- sentinel: v1.4.2
- adapter-framework: v2.0.0

Container Images (for HyperFleet Release 1.5):
- registry.ci.openshift.org/hyperfleet/api-service:v1.5.0
- registry.ci.openshift.org/hyperfleet/sentinel:v1.4.2
- registry.ci.openshift.org/hyperfleet/adapter-framework:v2.0.0

Compatibility:
- API Service v1.5.0 requires Adapter Framework ≥ v2.0.0
- Sentinel v1.4.2 is compatible with Adapter Framework v1.x and v2.x
- Full compatibility matrix documented in release notes
```

---

## 3. Release Readiness Criteria

Before declaring a release as "GA-Ready", all the following criteria must be satisfied:

### 3.1 Testing & Validation (Mandatory)

**Unit & Integration Testing:**
- ✓ All unit tests passing across all components
- ✓ Integration test suite passing

**E2E Testing:**
- ✓ Critical user workflows validated (E2E test suite)
- ✓ Backward compatibility testing with N-1 version
- ✓ Installation/upgrade path tested

**Performance & Load Testing:**
- ✓ Performance benchmarks show no regression > 10% vs. previous release
- ✓ Load testing validates system handles expected production load
- ✓ Resource utilization (CPU, memory) within acceptable bounds

**Note:** Automated testing is preferred for all scenarios. If a test scenario is not yet automated, it must be executed manually before release approval.

### 3.2 Bug Severity Gates (Mandatory)

- ✓ No open bugs with severity **Normal** or above (Blocker, Critical, Major, Normal)
    - Note: `Normal` bugs do not gate **MVP releases**
- ✓ Minor bugs:  No gate, tracked for future releases

### 3.3 Documentation Completeness (Mandatory)

**Release Documentation:**
- ✓ Release notes finalized (what's new, bug fixes, breaking changes)
- ✓ Known issues and limitations documented
- ✓ Deprecation notices published (if applicable)

**Operational Documentation:**
- ✓ Installation guide updated
- ✓ Upgrade instructions complete (N-1 → N)
- ✓ Deployment runbook created and reviewed

**Technical Documentation:**
- ✓ API documentation updated (if APIs changed)
- ✓ Configuration changes documented

### 3.4 Cross-Team Coordination

- ⚠ Integration validation with dependent offerings (TBD)

### 3.5 Security & Compliance

**Mandatory (All Phases):**
- ✓ Vulnerability scanning: No CRITICAL/HIGH CVEs in container images

**Post-MVP (Deferred to Konflux Migration):**
- ⚠ Supply chain security: SLSA Level 3 provenance generated
- ⚠ Software Bill of Materials (SBOM) generated
- ⚠ Container images signed with Sigstore/Cosign
- ⚠ Enterprise Contract Policy enforcement

**Note:** For MVP, focus on vulnerability scanning. Full supply chain security (SBOM, signing, provenance) will be implemented during Post-MVP Konflux migration.

### 3.6 Release Artifacts Verification (Mandatory)

- ✓ All container images built and pushed to registry
- ✓ Helm charts packaged and tested
- ✓ Git tags created with correct version
- ✓ Release artifacts checksums/signatures verified

**Gate Decision:** Only when ALL mandatory criteria are met can the release be declared GA-ready and published.

---

## 4. Bug Handling Workflow After Code Freeze

### 4.1 Bug Triage Process

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

### 4.2 Decision Framework

**For Blocker/Critical bugs:**
1. **Immediate Action:** Developer assigned within 2 hours
2. **Fix & Test:** Root cause analysis, fix implementation, automated tests added
3. **Cherry-Pick:** Fix merged to `main` first, then cherry-picked to release branch
4. **New RC:** Cut new release candidate (e.g., v1.5.0-rc.3)
5. **Regression Testing:** Full test suite re-run
6. **Time Box:** If fix takes > 24 hours, consider release delay or degrading severity

**For Major bugs:**
1. **Before Code Freeze:** Major severity bugs must be fixed before GA release
   - **Release Owner Assessment:** Evaluate impact, risk, complexity, and timeline
   - **Fix & Include:** Implement fix and cherry-pick to release branch
   - **If Not Fixable in Timeline:**
     - Consider release delay to allow fix completion
     - OR downgrade severity if impact assessment justifies (requires stakeholder approval and documented rationale)
2. **During Code Freeze:** Major bugs must be either:
   - **Escalated to Critical:** With documented justification showing blocker-level impact to be included in the release
   - **Deferred to Next Patch:** Scheduled for the next patch release (e.g., v1.5.1) if impact does not warrant Critical escalation

**For Normal/Minor bugs:**
- Default: Defer to next patch release or next minor release
- Track in backlog for future releases

### 4.3 Post-Code Freeze PR Approval Process

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
   - Update release tracking issue
   - Notify stakeholders if fix delays GA timeline

### 4.4 Hotfix Workflow (Post-GA)

For bugs discovered after GA release:

```bash
# Example assumes Sentinel component (v1.4.x)
# Create hotfix branch from component release tag
git checkout -b hotfix-1.4.3 v1.4.2

# Make fix, test, commit
git commit -m "Fix critical bug in Sentinel component"

# Merge to release branch
git checkout release-1.4
git merge --no-ff hotfix-1.4.3

# Tag component patch release
git tag -a v1.4.3 -m "Patch release v1.4.3"
git push origin release-1.4 --tags

# Cherry-pick to main if applicable
git checkout main
git cherry-pick <commit-sha>
```

**Hotfix Release Timeline:**
- Blocker/Critical severity: Patch release within 48 hours
- Major severity: Patch release within 1 week

### 4.5 Release Recovery Strategy

**HyperFleet uses a roll-forward recovery strategy for MVP releases.**

#### 4.5.1 Roll-Forward (Primary Strategy)

When issues are discovered in a GA release, the default recovery path is to **fix forward** via a patch release:

**Process:**
1. Identify and fix the issue in `main` branch
2. Cherry-pick fix to affected release branch
3. Cut patch release (e.g., v1.5.0 → v1.5.1)
4. Deploy patch following standard deployment procedures

**Timeline:**
- Blocker/Critical: Patch release within 48 hours
- Major: Patch release within 1 week

**Advantages:**
- Simpler testing scope (only test the fix)
- No database migration reversal complexity
- Maintains forward version progression
- Faster response for critical issues

#### 4.5.2 Rollback Support (Post-MVP)

**Status:** Deferred to Post-Q1 (separate epic required)

**Reference:** See [versioning trade-offs documentation](https://github.com/openshift-hyperfleet/architecture/blob/main/hyperfleet/docs/versioning-trade-offs.md#4-database-migration-and-rollback-procedures-post-mvp) for detailed rollback considerations.

**Decision Point:** Evaluate rollback support necessity after MVP based on:
- Incident frequency and severity
- Customer requirements for rollback capabilities
- Database schema stability
- Testing infrastructure maturity

---

## 5. Release Cadence

**Regular Releases:** Every 3 weeks (1 sprint) - ~17 releases per year
- Weeks 1-2: Active development
- Week 3: Feature freeze, stabilization, testing, GA release
  - Note: Duration may vary as CI/CD automation is being built

**Ad-Hoc Releases:** As needed for urgent requirements (3-5 days)
- Triggers: Critical bugs, security vulnerabilities, urgent business needs
- Reduced testing scope (unit, integration, E2E for affected components only)
- Release Owner approval required

**Cadence Refinement:**
As HyperFleet matures and based on retrospective data, the team may adjust the release cadence:
- If automation is strong and quality metrics support it, consider faster cycles
- If team coordination or integration complexity requires it, consider extending to 4 weeks
- Revisit cadence after every several releases during retrospectives

---

## 6. Release Artifacts and Deliverables

### 6.0 Release Repository

**Create: `openshift-hyperfleet/releases` repository**

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

### 6.1 Primary Release Artifacts

**Container Images:**
- Built automatically by Prow on release tag creation
- Published to `registry.ci.openshift.org/hyperfleet/*`
- **Image naming:** Each component uses its own independent semantic version
  ```text
  registry.ci.openshift.org/hyperfleet/{component}:v{component-version}
  ```
- **Example for HyperFleet Release 1.5:**
  - `registry.ci.openshift.org/hyperfleet/api-service:v1.5.0`
  - `registry.ci.openshift.org/hyperfleet/sentinel:v1.4.2`
  - `registry.ci.openshift.org/hyperfleet/adapter-framework:v2.0.0`

  (Each component has its own version reflecting its actual changes per Section 2.5)

**Helm Charts:**
- Each component has its own Helm chart in component repositories
  - Note: Adapter Framework base chart is designed for overlay usage by business adapters, not standalone deployment
- Note: Umbrella chart strategy (hyperfleet-chart repo) is under discussion

**Git Tags:**
- Git tags created independently in component repositories: `vX.Y.Z` (reflecting each component's version)
- HyperFleet Release tag created in `openshift-hyperfleet/releases` repository: `release-X.Y` (single source of truth for the validated component combination - see Section 6.0)
- GitHub Releases created from tags with release notes and compatibility matrix

### 6.2 Documentation Deliverables

#### 6.2.1 Release Notes

**Required Sections:**
```markdown
# HyperFleet Release 1.5

## Overview
Brief description of release theme and major highlights

## What's New
### New Features
- **API Service v1.5.0:** GitOps integration for ROSA deployments
- **Adapter Framework v2.0.0:** New Plugin API v2 with enhanced extensibility

### Enhancements
- **API Service v1.5.0:** OAuth 2.1 support, improved authentication flow
- **Sentinel v1.4.2:** Performance improvements in metrics collection

## Breaking Changes
- **Adapter Framework v2.0.0:** Plugin API v2 (migration guide: docs/plugin-migration-v2.md)
  - Requires business adapters to update to Plugin SDK v2.0+
- **API Service v1.5.0:** Deprecated `/v1/legacy-auth` endpoint removed

## Bug Fixes
- **Sentinel v1.4.2:** Fixed memory leak in metrics collector (#234)
- **Sentinel v1.4.2:** Fixed race condition in concurrent monitoring (#256)
- **API Service v1.5.0:** Resolved authentication token refresh issue (#198)

## Known Issues
- **Sentinel:** Metrics dashboard may show delay > 5 seconds under heavy load (workaround: increase polling interval)
- **API Service:** GitOps integration requires Kubernetes 1.28+ for full functionality

## Upgrade Instructions
See [Upgrade Guide](docs/upgrade-to-release-1.5.md) for detailed instructions.

## Compatibility Matrix

**HyperFleet Release 1.5** (validated component combination)

| Component | Version | Changes | Notes |
|-----------|---------|---------|-------|
| API Service | **v1.5.0** | MINOR | New GitOps integration, OAuth 2.1, breaking change (legacy auth removed) |
| Sentinel | **v1.4.2** | PATCH | Memory leak fix, performance improvements |
| Adapter Framework | **v2.0.0** | MAJOR | Plugin API v2 (breaking), enhanced extensibility |

**Component Compatibility:**
- API Service v1.5.0 requires Adapter Framework ≥ v2.0.0
- Sentinel v1.4.2 is compatible with Adapter Framework v1.x and v2.x
- Business adapters must upgrade to Plugin SDK v2.0+ to work with Adapter Framework v2.0.0

**Platform Compatibility:**

| Platform | Supported Versions |
|----------|-------------------|
| Kubernetes | 1.26 - 1.30 |
| Helm | 3.14+ |

## Security
- **All components:** Updated dependencies to address CVE-2026-1234, CVE-2026-5678
- **API Service v1.5.0:** Enhanced OAuth 2.1 security features
- **Sentinel v1.4.2:** No security-specific changes
```

**Checklist:**
- [ ] Release notes drafted during development
- [ ] Release notes finalized before GA
- [ ] Breaking changes clearly documented
- [ ] Known issues listed with workarounds
- [ ] Upgrade instructions validated
- [ ] Published to docs site and GitHub Release

#### 6.2.2 Upgrade/Installation Guide

**Content:**
- Prerequisites (Kubernetes version, permissions, dependencies)
- Fresh installation steps
- Upgrade path from N-1 version
- Adapter Framework: Deployment via business adapter overlay (not standalone)
- Post-installation validation steps
- Troubleshooting common issues

**Checklist:**
- [ ] Installation guide updated
- [ ] Upgrade path tested (N-1 → N)
- [ ] Screenshots/examples updated
- [ ] Published to documentation site

#### 6.2.3 API Documentation

**For API Service:**
- OpenAPI/Swagger specification updated
- API reference documentation published
- Code examples for new endpoints
- Deprecation notices for old APIs

**Checklist:**
- [ ] OpenAPI spec generated from code
- [ ] API docs published (e.g., via Swagger UI)
- [ ] Examples tested and validated
- [ ] Breaking changes highlighted

#### 6.2.4 Change Log

**Format:** Keep a Changelog standard (per component)

**API Service - CHANGELOG.md:**
```markdown
# Changelog - API Service

## [1.5.0] - 2026-05-12

### Added
- New GitOps integration for ROSA deployments
- OAuth 2.1 authentication support

### Changed
- Updated authentication flow to use OAuth 2.1
- Improved API response caching mechanism

### Removed
- `/v1/legacy-auth` endpoint (deprecated since v1.3.0)

### Fixed
- Authentication token refresh issue
- Race condition in concurrent API requests

### Security
- Updated dependencies to address CVE-2026-1234
```

**Sentinel - CHANGELOG.md:**
```markdown
# Changelog - Sentinel

## [1.4.2] - 2026-05-12

### Fixed
- Memory leak in metrics collector
- Race condition in concurrent monitoring

### Security
- Updated dependencies to address CVE-2026-5678
```

**Adapter Framework - CHANGELOG.md:**
```markdown
# Changelog - Adapter Framework

## [2.0.0] - 2026-05-12

### Added
- Plugin API v2 with enhanced extensibility
- Support for async plugin initialization

### Changed
- **BREAKING:** Plugin API v2 replaces v1 (migration guide: docs/plugin-migration-v2.md)
- Improved plugin loading performance by 40%

### Removed
- **BREAKING:** Plugin API v1 support

### Security
- Updated dependencies to address CVE-2026-1234
```

**Checklist:**
- [ ] CHANGELOG.md updated in repository
- [ ] All significant changes categorized
- [ ] Links to PRs/issues included
- [ ] Security fixes clearly marked

### 6.3 Compliance and Security Artifacts

**MVP:**
- Vulnerability scanning of container images
- No CRITICAL/HIGH CVEs in release

**Post-MVP (with Konflux):**
- Enterprise Contract Policy enforcement
- SBOM generation
- SLSA Level 3 provenance
- Image signing with Sigstore/Cosign

---

## 7. Konflux vs. Prow Comparison

### 7.1 Current State: Prow

**What Works:**
- Automated CI/CD pipeline (testing, image builds)
- GitHub integration
- Team familiarity

**What's Missing:**
- SLSA provenance, SBOM generation, image signing
- Requires additional tooling for supply chain security

### 7.2 Konflux Benefits

**Supply Chain Security:**
- SLSA Level 3 provenance, SBOM, Sigstore signing (built-in)
- **Enterprise Contract Policy enforcement**
- Integrated vulnerability scanning

**Release Automation:**
- Unified build-test-release workflow
- OCI artifact management
- Policy-as-code compliance gates

### 7.3 Recommendation

**MVP Approach:**
- **Use Prow + manual release process**
- Manual steps: branching, tagging, Helm packaging, GitHub Releases
- Defer security tooling to Post-MVP
- Focus: Establish process first, automate later

**Post-MVP Migration:**
- Migrate to Konflux for automated releases
- Add Enterprise Contract Policy enforcement
- Implement SBOM, provenance, signing automation

---

## 8. Next Steps

### 8.1 MVP Tickets (First Release Preparation)

**[TICKET-1] Create HyperFleet Releases Repository**
- **Objective:** Set up `openshift-hyperfleet/releases` as single source of truth
- **Tasks:**
  - Initialize repository structure (release notes, docs, charts)
  - Set up issue templates for release tracking and ad-hoc requests
  - Document repository purpose and usage in README
- **Reference:** Section 6

**[TICKET-2] Establish Release Cadence and Calendar**
- **Objective:** Define release schedule to meet first release
- **Tasks:**
  - Decide first release date and version number
  - Publish release calendar for next 3 months (including first release)
  - Document ad-hoc release criteria and process
  - Communicate calendar to team and stakeholders
- **Reference:** Section 5

**[TICKET-3] Prepare for First Release**
- **Objective:** Document procedures and assign ownership for first release
- **Tasks:**
  - Write runbook: Release branching, tagging, image promotion
  - Write runbook: Helm chart packaging and GitHub Release creation
  - Finalize release checklist
  - Assign Release Owner for first release
  - Document Release Owner responsibilities (gatekeeper, approver)
  - Plan rotation strategy for future releases
- **Reference:** Sections 2, 3, 6

**[TICKET-4] Execute First Release**
- **Objective:** Run first release following documented process
- **Tasks:**
  - Create `release-v{X.Y}` branch at feature freeze
  - Cut Release Candidate (RC.1)
  - Execute testing per Section 3.1
  - Cherry-pick bug fixes if needed (follow Section 2 process)
  - Cut final GA release
  - Publish release artifacts and documentation
  - Conduct post-release retrospective
- **Reference:** Sections 2, 3, 6

### 8.2 Post-MVP Improvements

#### 8.2.1 Conduct Retrospectives and Identify Improvements

After completing the first few releases with manual processes, conduct retrospectives to:
- Identify workflow pain points and bottlenecks
- Determine which manual steps should be automated
- Evaluate release process effectiveness (timing, quality gates, coordination)
- Gather feedback from Release Owners, developers, and stakeholders
- Update release procedures based on lessons learned
- Prioritize automation opportunities (Helm packaging, release notes generation, GitHub Releases)

#### 8.2.2 Migrate to Konflux for Official Releases

Transition from manual Prow-based releases to Konflux for production-grade, compliant releases:

##### Why Konflux

- **Enterprise Contract Policy** enforcement for compliance and security gates
- Makes releases more official with built-in governance
- SLSA Level 3 compliance (provenance, SBOM, signing)
- Unified build-test-release pipeline with policy-as-code

##### Migration Approach

- Evaluate Konflux with pilot project (test environment, parallel builds with Prow)
- Implement Enterprise Contract Policy framework and define policies
- Migrate all components to Konflux pipelines
- Automate SBOM generation, image signing, and provenance
- Full cutover after validation

#### 8.2.3 Additional Process Improvements

Based on retrospective findings and Konflux capabilities:
- Establish automated E2E test gate as mandatory release criteria
- Create release health monitoring dashboards
- Define SLI/SLO framework for release quality metrics
- Optimize release cadence based on data (6-month review)
- Consider LTS release designation (e.g., every 4th release)

### 8.3 Success Metrics

**Track and review quarterly:**

**Release Metrics:**
- HyperFleet Release frequency (target: ~17 releases/year with 3-week cadence)
- Component patch release frequency (individual component updates between HyperFleet Releases)
- Code freeze duration (target: < 1 week, ideally 3-4 days)
- On-time delivery (target: > 80% of HyperFleet Releases on schedule)
- Stabilization phase variance (track actual vs. planned to inform process improvements)

**Quality Metrics:**
- Bug escape rate (bugs found post-GA per component and per HyperFleet Release)
- Hotfix frequency per component (target: < 2 patch releases per component between HyperFleet Releases)
- Mean time to patch critical vulnerabilities (target: < 48 hours for any component)
- Cross-component compatibility issues found in production (target: 0)

---

## 9. Appendices

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

- **Feature Freeze:** Deadline after which no new features accepted for current release
- **Code Freeze:** Period when only critical bug fixes are allowed into release branch
- **GA (General Availability):** Official release available to all users
- **RC (Release Candidate):** Pre-release version for final testing
- **SLSA:** Supply-chain Levels for Software Artifacts - security framework
- **SBOM:** Software Bill of Materials - list of all components in software
- **Cherry-Pick:** Applying specific commits from one branch to another
- **Hotfix:** Urgent fix applied to released version outside normal release cycle
- **LTS:** Long-Term Support - release with extended maintenance period
- **N-1 Compatibility:** Supporting one version back (e.g., v1.5 compatible with v1.4)

### Appendix C: Template - Release Tracking Issue

```markdown
# HyperFleet Release 1.5 Tracking Issue

## Timeline (3-week sprint cycle)
- Sprint Start: YYYY-MM-DD
- Feature Freeze: YYYY-MM-DD
- Code Freeze: YYYY-MM-DD
- GA Target: YYYY-MM-DD

**Note:** Dates may shift based on stabilization phase needs as automation is being built.

## Release Owner
@username

## Component Versions for Release 1.5

| Component | Target Version | Status | Notes |
|-----------|---------------|--------|-------|
| API Service | v1.5.0 | 🟡 In Progress | GitOps integration, OAuth 2.1 |
| Sentinel | v1.4.2 | 🟢 Ready | Bug fixes only |
| Adapter Framework | v2.0.0 | 🟡 In Progress | Breaking: Plugin API v2 |

## Release Criteria Status
- [ ] All planned features complete
- [ ] E2E tests passing with component combination
- [ ] No Blocker/Critical/Major bugs in any component
- [ ] Cross-component compatibility validated
- [ ] Documentation complete (including compatibility matrix)
- [ ] Pillar team sign-off

## Release Candidates
- [ ] v1.5.0-rc.1 (April 15, at Feature Freeze)
- [ ] v1.5.0-rc.2 (April 16-17, if needed)
- [ ] v1.5.0-rc.3 (April 18, if needed)

## Blockers
- None currently

## Compatibility Validation
- [ ] API Service v1.5.0 + Adapter Framework v2.0.0 integration tested
- [ ] Sentinel v1.4.2 compatibility with all components verified
- [ ] Backward compatibility with Release 1.4 validated
- [ ] Breaking changes documented with migration guides

## Communication
- [ ] Release notes drafted (including compatibility matrix)
- [ ] Breaking changes highlighted
- [ ] Stakeholders notified (T-1 week)
- [ ] Release announcement prepared

## Post-Release
- [ ] Retrospective scheduled
- [ ] Metrics collected
- [ ] Stabilization phase variance documented
- [ ] Component version tracking updated
```

### Appendix D: Template - Ad-Hoc Release Request

```markdown
# Ad-Hoc Release Request: HyperFleet Release 1.5.1 (or Component-Specific Patch)

## Release Type
- [ ] **Full HyperFleet Release** (multiple components, validated combination)
- [ ] **Single Component Patch** (e.g., Sentinel v1.4.3 only, no new HyperFleet Release number)

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
| API Service | v1.5.0 | v1.5.1 | PATCH | Critical security fix |
| Sentinel | v1.4.2 | v1.4.2 | No change | - |
| Adapter Framework | v2.0.0 | v2.0.0 | No change | - |

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
- [ ] API Service - [changes description]
- [ ] Sentinel - [changes description]
- [ ] Adapter Framework - [changes description]

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
- [ ] Fix timeline estimated (target: < 48 hours for critical)
- [ ] Workaround available for users (if applicable)

### Database Migration Considerations
- [ ] Schema changes included? [yes/no]

## Stakeholder Coordination

### Offering Team Notification
- [ ] Offering team notified (minimum 48 hours advance)
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

| Day | Date | Activities | Owner |
|-----|------|-----------|-------|
| 1 | YYYY-MM-DD | Development + unit tests | @dev-team |
| 2 | YYYY-MM-DD | Code review + CI tests | @reviewers |
| 3 | YYYY-MM-DD | E2E testing + RC build | @qa-team |
| 4 | YYYY-MM-DD | Manual testing + stakeholder review | @qa + @stakeholders |
| 5 | YYYY-MM-DD | GA release + deployment | @Technical Lead and @Manager |

## Post-Release Monitoring
- [ ] Metrics dashboard monitored for 24 hours
- [ ] No error rate increase
- [ ] No performance degradation
- [ ] Post-release review completed
```

---
