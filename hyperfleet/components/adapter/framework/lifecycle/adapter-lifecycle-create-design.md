---
Status: Active
Owner: HyperFleet Team
Last Updated: 2026-07-09
---

# Adapter Lifecycle: Conditional Creation

**Jira**: [HYPERFLEET-1295](https://issues.redhat.com/browse/HYPERFLEET-1295)

## What & Why

**What**: `lifecycle.create.when` is an optional per-resource CEL gate on a resource's *initial* creation. When the resource does not yet exist and the expression evaluates to `false`, the resource is skipped instead of created.

**Why**: Preconditions gate the entire resources phase — all or nothing. That forces adapter authors to either write one coarse precondition covering every resource, or duplicate config across multiple task configs to get per-resource conditional behavior. `lifecycle.create.when` lets a single resource opt out of creation based on runtime state (a feature-flag param, a sibling resource's discovered state, an event field) without blocking the rest of the resources phase. Part of the broader adapter DSL improvement effort (HYPERFLEET-1174).

**Related Documentation:**

- [Adapter Lifecycle: Deletion](./adapter-lifecycle-delete-design.md) — the symmetric `lifecycle.delete.when` design this mirrors
- [Adapter Framework Design](../adapter-frame-design.md) — current framework architecture

## DSL

```yaml
resources:
  - name: optionalFeatureConfig
    manifest: ...
    discovery:                 # required: needed to check whether the resource already exists
      by_name: "{{ .clusterId }}-feature"
    lifecycle:
      create:
        when:
          expression: "params.?enableOptionalFeature.orValue(false)"
```

**Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `lifecycle.create.when.expression` | **YES**, when `lifecycle.create.when` is configured | string (CEL) | Evaluated only while the resource doesn't yet exist. Required — the adapter validator rejects configs where `lifecycle.create` is present but `when` is absent or empty. |

- `discovery` must be configured on the same resource — without it the executor cannot determine whether the resource already exists, and the validator rejects the config.
- `lifecycle.create` is entirely optional; a resource without it is applied unconditionally (backward compatible, existing behavior).

## Semantics

- **Resource doesn't exist yet**: `when` is evaluated. `false` → skip creation (`adapter.resourcesSkipped` set to `true`, logged at INFO). `true`, or no `lifecycle.create` configured at all → proceed with normal create.
- **Resource already exists**: `when` is **ignored** — the resource is always applied normally (update flow). This makes the gate a one-time check on initial creation, not a recurring condition re-evaluated every reconciliation; once created, the resource behaves like any other.
- **CEL evaluation error**: execution fails with a descriptive error naming the failing expression. The resource is never silently skipped or applied on error — same fail-closed behavior as `lifecycle.delete.when`.
- Resources are pre-discovered before any `lifecycle.create.when`/`lifecycle.delete.when` is evaluated, so a sibling-dependency expression like `resources.?resourceA.hasValue()` reflects state from prior reconciliations regardless of list order.

## Migration & Compatibility

`lifecycle.create` is optional and additive:

- Not set → existing apply-only behavior, unchanged.
- Set without `when` → config validation rejects it (mirrors `lifecycle.delete`'s requirement).

No API, Sentinel, or status-contract changes required — this is purely an adapter-framework DSL and executor change, scoped to the resources phase.

## References

- [HYPERFLEET-1295](https://issues.redhat.com/browse/HYPERFLEET-1295) — Resource-level `when` condition for conditional creation
- [HYPERFLEET-1174](https://issues.redhat.com/browse/HYPERFLEET-1174) — Adapter DSL improvement effort
- [Adapter Lifecycle: Deletion](./adapter-lifecycle-delete-design.md)
