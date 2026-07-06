---
Status: Active
Owner: HyperFleet QE Team
Last Updated: 2026-06-09
---

# Test Placement Strategy

Where to put a new test, and why. Applies to every HyperFleet component — API, Sentinel, adapters, broker — and every resource type (clusters, nodepools, and any future resources).

## The Rule

**Test at the lowest layer that can catch the failure mode.**

| Failure Mode | Test Layer |
|---|---|
| Pure validation, error codes, business logic with no I/O | Unit |
| Serialization, middleware, real DB constraints, query behavior | Integration |
| Multi-component interaction (events propagate, adapters execute, K8s resources appear, status aggregates across services) | E2E |
| Deployment reality (shipped container images, K8s manifests, TLS/identity configuration, contract validation enabled) | E2E |

A test belongs in E2E **only** when the failure requires multiple deployed components — or the real deployed environment — to manifest. If the bug could be caught with a mock or a real DB within a single service, it does not belong in E2E.

## What E2E Is and Isn't

E2E tests verify **user journeys**: the behavior of the whole system from the perspective of a user's actions. A journey is a sequence the user would recognize — "I created a cluster and it became ready," "I deleted it and everything downstream was cleaned up." Every E2E test should map to a journey like this; if you can't name the journey, the test probably belongs at a lower layer.

E2E tests do not re-test CRUD. The API's ability to accept a POST, store a row, and return it on GET is a component/service concern — each service repo's own unit and integration tests cover it.

E2E tests exist to verify **what happens after the API responds**: Sentinel picks up the event, dispatches to adapters, adapters execute against K8s, report status back, status aggregates into top-level conditions, and the resource reaches its final state. The "create" call in an E2E test is the start of the journey, not the thing being tested.

**The line:** if the assertion is about the HTTP response (status code, field presence, error shape), it belongs in a service repo. If the assertion is about what happened downstream (adapter executed, K8s resource exists, condition propagated), it belongs in E2E. A journey passes through API calls and the test fails if those calls fail — but the assertions that justify the test's existence are about the journey's outcome, not response schemas. Response shape is already gated at the integration layer.

Examples across resource types:

| Assertion | Layer | Why |
|---|---|---|
| POST returns 201 with correct ID format | Integration | HTTP response shape — single service |
| POST with duplicate name returns 409 | Integration | DB unique constraint — single service |
| Created resource reaches Reconciled=True | E2E | Requires Sentinel + adapters + status aggregation |
| PATCH increments generation, no-op on same spec | Unit | Service logic — mock DAO |
| PATCH triggers adapter re-reconciliation at new generation | E2E | Requires adapters to observe the generation change |
| DELETE returns 202 with deleted_time set | Integration | HTTP response shape — single service |
| DELETE cascades soft-delete to child resources in DB | Unit + Integration | Service logic + DB transaction |
| DELETE triggers adapter finalization → hard-delete → K8s cleanup | E2E | Full pipeline across components |
| Adapter crash → resource stuck → restore → reconciled | E2E | Infrastructure failure + recovery across components |

These examples use API resource operations, but the rule is component-agnostic: adapter, Sentinel, and broker tests follow the same logic — place each failure at the lowest layer that can manifest it.

## Real Conditions

E2E is the only layer that runs the system as it actually ships:

- Components deployed to Kubernetes from the real manifests
- Real containerized binaries — the images we ship, not `go test` processes
- Production-like configuration: identity/TLS enabled, OpenAPI (`openapi.yaml`) contract validation enforced

This makes deployment reality a failure-mode category of its own (the fourth row in the table above). A broken image build, manifest or env-var drift, a TLS misconfiguration, or a validation flag that's on in production but off in dev will pass every unit and integration test and fail only here. These don't need multiple components to be broken — they need the real environment to manifest.

## Beyond Regression Catching

The journey suite also serves as external evidence:

- **Proof for partners.** When a partner asks whether an action they depend on is covered, the answer is a link to a named journey test, not an assurance. Keep journey names legible to people outside the team for exactly this reason.
- **Release verification.** The suite runs against any deployed release. During a production incident, re-running the journeys against the affected release narrows the search: a journey that passes in E2E but fails in production points at an environmental difference — and the realistic E2E setup keeps that diff small.
- **Living showcase.** The journey list doubles as executable documentation of what the HyperFleet system does end to end.

None of these roles justify duplicating lower-layer assertions in E2E. They're properties the journey tests already have when placed by the rule.

## Intentional Overlap vs. Redundancy

Two tests covering the same behavior at different layers is **not** redundancy when they catch different failure modes.

Take soft-delete as an example (applies to any resource type):

- **Unit** catches: logic bug in cascade ordering, idempotency check, generation increment.
- **Integration** catches: HTTP 202 response shape, `deleted_by` populated from JWT, DB transaction commits correctly.
- **E2E** catches: adapters actually finalize, Sentinel triggers hard-delete, downstream resources cleaned up.

Same behavior, three failure modes. This is the pyramid working.

**Redundancy** is when a higher-layer test catches the exact same failure mode as a lower-layer test. Examples:

- An E2E test that validates GET response fields (id, name, generation) after creation — the creation test already GETs and validates, and integration tests cover the response schema.
- An E2E test that checks LIST pagination — this is a query concern fully covered by integration tests.

Delete the redundant copy. Keep the intentional overlap.

## Layer Definitions

### Service Repos

Every HyperFleet service repo owns its own unit and integration tests, testing that service in isolation — no other HyperFleet components running. The list of service repos grows as services are added; the testing principle stays the same.

#### Unit tests (e.g., `pkg/services/`, `pkg/handlers/`)

Runs against mock dependencies. No HTTP server, no database, no external services.

**What belongs here:**

- Input validation (name format, kind, spec, condition status values)
- Service-layer state machines (condition transitions, stale update policy, timestamp stability)
- Mutation logic (soft-delete cascade, generation increment, idempotency)
- Error mapping (service errors → HTTP status codes)
- Semantic equality checks (no-op detection for patches)

#### Integration tests (e.g., `test/integration/`)

Runs a real HTTP server against a real database. Single component — no other HyperFleet services running. The exact integration surface depends on what each service talks to (e.g., the API tests against a real DB; an adapter might test against a mock broker).

**What belongs here:**

- HTTP response shape (field presence, types, href format, error details)
- Auth middleware (401/403 for missing or invalid tokens)
- DB constraints (unique names, foreign keys, `SELECT FOR UPDATE` under concurrency)
- Query behavior (pagination, search, sorting, filtering, soft-deleted row visibility)
- Schema validation (OpenAPI middleware rejects invalid payloads)
- Status processing (adapter status reports trigger condition aggregation and hard-delete — the API processes these directly, no Sentinel needed)

### E2E (`hyperfleet-e2e`)

A separate repository from all service repos. Tests run against the full deployed stack — API, Sentinel, adapters, K8s cluster — deployed from the real manifests, running the shipped container images under production-like configuration (identity/TLS, contract validation enabled).

**What belongs here — user journeys across all resource types:**

- Create → adapters execute → downstream resources created → Reconciled
- Update → adapters re-reconcile at new generation → coalescing
- Delete → adapters finalize → hard-delete → downstream cleanup
- Cascade operations through the full pipeline (parent delete cascades to children through adapter finalization, not just DB)
- Adapter dependency ordering (workflow engine enforces execution order)
- Conflicting operations (delete during reconciliation, update during deletion)
- Resource re-creation after hard-delete (name freed by full lifecycle, not just DB delete)
- External resource deletion (adapters finalize cleanly despite missing downstream resources)
- Adapter crash recovery (unavailable → resource stuck → restore → reconciled)
- Non-required adapter failure isolation (resource reconciles despite optional adapter failure)
- Stuck finalization (adapter unable to finalize blocks hard-delete → restore or force-delete unblocks)
- Concurrent operations with resource isolation

## Decision Checklist

Before writing a test, answer these questions in order:

1. **Can I catch this with a mock?** Write a unit test. Stop.
2. **Does it need real HTTP + real DB but only one service?** Write an integration test. Stop.
3. **Does it require multiple deployed components, or the real shipped artifacts and configuration?** Write an E2E test — and name the user journey it verifies.

If you're unsure, ask: **"What would have to be broken for this test to fail?"**

- "The service function has a logic bug" → unit.
- "The DB doesn't enforce the constraint" or "the middleware doesn't wire correctly" → integration.
- "Sentinel didn't dispatch the event" or "the adapter didn't execute" or "the K8s resource wasn't created" → E2E.
- "The shipped image, manifest, or production configuration is wrong" → E2E.

## Coverage Durability

Tests only protect against regressions when they **gate merges**. A test that exists but doesn't block CI is documentation, not a safety net.

**Required controls:**

- Every repo must have its test jobs listed as **required status checks** in branch protection. No merging past red.
- Test directories must have **CODEOWNERS** entries. Deleting a test file requires reviewer approval from the owning team.
- When deleting a test, the PR description must state which other test covers the same failure mode and at which layer. Reviewers verify the claim.
- When moving a test to a lower layer: add the new test, verify it's green **and gated** in CI, then delete the old one. Never leave a window where coverage exists but doesn't block merges.
