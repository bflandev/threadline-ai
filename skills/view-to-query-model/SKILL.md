---
name: view-to-query-model
description: >
  Use this skill when the user has view specs (V-XX) and domain events defined,
  and needs to derive read model specifications for the CQRS query side. Trigger
  when the conversation references read models, projections, query models,
  denormalised views, or asks how a view gets its data from domain events.
---

# View to Query Model

## Overview

In CQRS, aggregates are optimised for enforcing invariants, not serving screens. Querying them directly couples read performance to write-side structure and blocks UI evolution on domain model changes.

A read model is a **projection** — it consumes domain events and maintains a denormalised, query-optimised representation shaped for one view. Each read model serves a specific view spec, updated asynchronously as events occur.

## Dependencies

This skill requires two prior outputs:

- **View specs** (from `view-spec-writer`) — the V-XX identifiers, field lists, filters, sort orders, and interaction patterns that define what data a screen needs.
- **Domain events** (from `use-case-to-technical-design`) — the events raised by aggregates that carry the data the read model will consume.

## Derivation Steps

1. **Start from the view spec.** Take V-XX and list every field, filter, and sort the view requires.
2. **Identify source events.** For each field, trace back to the domain event(s) that create or change that data. A single read model typically consumes events from multiple aggregates.
3. **Define the schema.** Map each view field to a read model field with its type and origin (`aggregate.field` or computed from event data).
4. **Determine indexes.** Every field used in a filter or sort in the view spec needs an index on the read model. Compound indexes for multi-field queries.
5. **Set staleness tolerance.** Match the view's UX requirements: real-time for dashboards users stare at, eventual (seconds) for lists, batch (minutes) for reports.

## Read Model Template

```
Read Model: RM-XX
Serves: V-XX [view spec ID]
Source events: [domain events that update this read model]
Schema:
  - [field name]: [type] -- derived from [aggregate.field]
  - [field name]: [type] -- computed from [event data]
Indexes: [fields that need indexes for the query patterns in the view spec]
Staleness tolerance: [real-time | eventual -- N seconds | batch -- N minutes]
```

Number read models to match their view spec: RM-01 serves V-01, RM-02 serves V-02, and so on.

## Integration Test Pattern

Read model tests verify that consuming a sequence of domain events produces the expected projection state. Name tests after the view spec they serve.

```
Test: RM-01 for V-01
Given events:
  1. TableCreated { id: "t1", name: "Main Hall - 4", capacity: 4 }
  2. TableStatusChanged { id: "t1", status: "occupied" }
Then read model state:
  - tables: [{ id: "t1", name: "Main Hall - 4", capacity: 4, status: "occupied" }]
```

Replay events from an empty state. Test event ordering, idempotency, and gap handling.

## Output Format

For each view spec, produce one read model specification using the template above, followed by at least one integration test scenario.

### Verification Checklist

- [ ] Every field in the view spec maps to a read model schema field
- [ ] Every source event is a real domain event from the technical design
- [ ] Every filter/sort field in the view spec has a corresponding index
- [ ] Staleness tolerance matches the view's UX requirements
- [ ] Integration test covers the primary happy-path event sequence
- [ ] No direct queries against aggregates — all reads go through the read model

## Common Mistakes

- **Querying aggregates directly.** If you find yourself joining aggregate tables for a view, you need a projection instead.
- **Missing indexes.** Every filter and sort in the view spec implies a query pattern. No index means a table scan.
- **Wrong staleness tolerance.** Real-time for a monthly report wastes infrastructure. Batch for a live status board frustrates users.
- **One read model for many views.** Each view spec gets its own read model. Sharing projections reintroduces the coupling CQRS eliminates.
- **Forgetting computed fields.** Some view fields are derived from event data (counts, durations, flags), not stored on any aggregate. Map these explicitly.
