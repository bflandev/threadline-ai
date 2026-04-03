# Threadline v2: Gaps, Friction, and the Road Forward

## An Honest Assessment of the Methodology and What Comes Next

---

## Part One: What Works

Before cataloguing what's missing, it's worth being precise about what Threadline already does well — because these strengths define the foundation that v2 builds on, not replaces.

### The Use Case Format

The fully dressed use case format is Cockburn's design, battle-tested across two decades of software projects. It works because it forces completeness: preconditions, extensions, and observable postconditions are not optional fields — they are the fields where the majority of real-world bugs and design gaps originate. Teams that adopt the format consistently report that the most valuable artefact is the extension list, because it is the first time anyone has systematically asked "what can go wrong at each step?" before writing code.

### The Mechanical Derivation

The strongest part of Threadline is the deterministic translation from use case elements to downstream artefacts. Preconditions become invariant guards. Extensions become typed errors. Postconditions become test assertions. Main success scenario steps become interactor method bodies. This is not a loose analogy — it is a mechanical mapping that produces the same architecture from the same use case every time. The mapping is what makes the pipeline automatable and the output trustworthy.

### The TDD-Aware AI Prompts

The instruction "make these pre-existing tests pass, do not write your own" solves a real and well-documented problem with AI-assisted code generation. Without pre-existing tests, the AI generates code and tests together, optimising for tests that pass rather than tests that correctly specify behaviour. Threadline's approach — derive the tests from the use case, then constrain the AI with those tests — closes this gap. It is the single most practically valuable idea in the methodology for teams already using AI code generation.

### The Skill-Based Automation

Each of the ten skills is scoped tightly enough that a current-generation AI agent can execute it reliably in a single invocation. The skills are read at the point of invocation rather than loaded into context at startup, which prevents context dilution. The pipeline DAG with explicit gates is a clean coordination model. Teams have successfully run partial pipelines (use cases → tests only, use cases → design only) without the full ceremony.

### The Bidirectional Traceability

The naming conventions — test names referencing use case IDs, screen IDs referencing step numbers, Figma annotations referencing extension IDs, code comments referencing step numbers — collectively create a chain that can be followed in both directions. Forward from requirement to code, backward from bug to requirement. This chain is the core value proposition and it holds up in practice for command-side features.

---

## Part Two: What's Missing

The gaps identified here are not structural flaws in Threadline. They are areas the methodology does not yet cover that real teams will encounter in the first week of adoption. Each gap is described with the problem it creates, the situations where it matters most, and the proposed v2 solution.

---

### Gap 1: The Query Side

**The problem.** Threadline has a command-side bias. It handles "user does X, system responds with Y" beautifully — form submissions, record creation, workflow transitions, permission changes. But most products are 70% reads. A dashboard, a search results page, a reporting view, a filtered table, a notification feed — these do not have a meaningful "main success scenario" beyond "user opens the page, system shows data." Forcing a dashboard into the fully dressed use case format produces either a trivially short use case (three steps, no meaningful extensions, no preconditions worth stating) or a grotesquely inflated one where the extensions are all display-logic edge cases that don't belong in a specification document.

**When it matters.** Any product with a significant read layer. SaaS dashboards, analytics products, content platforms, admin panels. The FieldBase CMS worked example is dominated by command-side use cases (create table, submit form, build page), but a real implementation would spend most of its development time on the table view, the record detail view, the page renderer, and the form builder UI — all read-heavy.

**What happens without a solution.** Teams either force read features through the use case pipeline (producing awkward, low-value specs) or skip Threadline entirely for read features (losing traceability for the majority of the product). Both outcomes undermine the methodology.

**v2 proposal: View Specifications.**

Introduce a lightweight companion format for read-heavy features:

```
View ID: V-XX
Title: [what the user sees]
Actor: [who sees it]
Data source: [which aggregate(s) or read model(s)]
Trigger: [what causes this view to appear — a route, a navigation action, a state change]

Display rules:
  1. [What is shown and how — declarative, not procedural]
  2. [Sorting, filtering, grouping, pagination defaults]
  3. [What happens when the data set is empty]

Conditional display:
  Ca. [Condition] → [what changes in the display]
  Cb. [Condition] → [what changes in the display]

Performance constraints:
  - [Target load time, pagination size, caching strategy if relevant]
```

View specifications are simpler than use cases but maintain the Threadline conventions: IDs that propagate into screen inventory, conditional display rules that map to component variants, empty data states that map to empty state screens.

The pipeline DAG gains a fourth branch in Phase 3: Branch D runs `view-spec-writer` → `view-to-query-model` → feeds into the screen inventory alongside use case screens. View specs do not go through the TDD pipeline (they are tested with integration tests against the read model, not the four-level suite), and they do not produce AI implementation prompts (query implementations are typically straightforward enough that the prompt adds no value).

The `screen-inventory` skill is updated to accept both use cases and view specs as input. Screen IDs from view specs follow the pattern S-V-[view number]-[sequence].

---

### Gap 2: Non-Functional Requirements

**The problem.** Performance, scalability, security hardening, rate limiting, caching, observability, compliance, data retention — these are mentioned once in Threadline as "stakeholders & interests" and then effectively disappear from the pipeline. But non-functional requirements drive enormous architectural decisions. A use case that says "system writes a new record" tells you nothing about whether that write needs to handle 10 requests/second or 10,000. A precondition that says "user has editor role" tells you nothing about whether the auth system needs to support SSO, MFA, or role delegation.

**When it matters.** Any product with performance-sensitive operations, regulatory requirements, multi-tenant isolation, or SLAs. Also any product where the non-functional requirements differ significantly across use cases — a public form submission has very different performance and security profiles than an internal admin operation.

**What happens without a solution.** Non-functional requirements are handled ad hoc by individual developers at the adapter layer. Decisions are made inconsistently, constraints are not documented, and when a performance problem surfaces there is no traceable requirement to point to.

**v2 proposal: Quality Attribute Scenarios.**

Introduce a supplementary artefact that attaches non-functional requirements to specific use cases or to the system as a whole. The format is adapted from the SEI quality attribute scenario template:

```
QA-XX: [Short name]
Applies to: UC-XX (or "system-wide")
Attribute: [performance | security | scalability | availability | compliance | observability]
Stimulus: [What triggers the concern — "1000 concurrent form submissions", "user attempts SQL injection"]
Response: [What the system must do — "process within 200ms p95", "reject and log the attempt"]
Measure: [How to verify — "load test with k6", "OWASP ZAP scan passes"]
```

Quality attribute scenarios feed into the adapter and infrastructure layers of the technical design. The `use-case-to-technical-design` skill is updated to accept QA scenarios as additional input and to annotate adapter and infrastructure decisions with the QA ID that requires them. Integration tests gain a new category: performance and security contract tests derived from QA scenarios.

Quality attribute scenarios do not go through the review gate (they are reviewed alongside the technical design at Gate 2) and do not affect the design pipeline.

---

### Gap 3: Cross-Cutting Concerns

**The problem.** Authentication, authorization beyond field-level access, audit logging, multi-tenancy, feature flags, rate limiting, request tracing, error reporting — these span all use cases and do not map cleanly to any individual one. The bounded context model handles some of this (the Identity & Access shared kernel), but the Threadline pipeline has no skill for deriving middleware, auth guards, tenant isolation, or observability infrastructure from the specification.

**When it matters.** Any product with more than one user role, more than one tenant, or any compliance requirement. In practice, this means every product.

**What happens without a solution.** Cross-cutting concerns are implemented as framework middleware or decorators by individual developers, with no traceability to any requirement. When the auth model changes, there is no systematic way to identify which use cases are affected. When an audit requirement is added, there is no way to verify that all relevant use cases are covered.

**v2 proposal: Cross-Cutting Concern Registry.**

Introduce a registry that maps cross-cutting concerns to the use cases they apply to:

```
Cross-Cutting Concern: Authentication
Applies to: All use cases except UC-02 (public form submission)
Implementation: Middleware on all routes; UC-02 routes excluded
Derived from: Actor declarations — use cases with "authenticated user" require auth

Cross-Cutting Concern: Audit Logging
Applies to: UC-01 (table creation), UC-04 (field access changes)
Implementation: Domain event handler — listens for TableCreated, FieldAccessChanged
Derived from: Stakeholder interest — compliance requires change tracking

Cross-Cutting Concern: Rate Limiting
Applies to: UC-02 (form submission — public endpoint)
Implementation: Adapter-layer middleware on /forms/*/submissions
Derived from: QA-03 (see quality attribute scenarios)
```

The registry is produced after Phase 2 (once all use cases are reviewed and all actors are known) and is reviewed at Gate 2 alongside the technical design. The `use-case-to-technical-design` skill is updated to produce the registry as an additional output, derived from actor declarations, stakeholder interests, and quality attribute scenarios.

---

### Gap 4: The Read Model and CQRS

**The problem.** The architecture section covers aggregates, interactors, and ports in depth — all command-side patterns. But there is no coverage of projections, read models, or how to serve the data views that most of the UI actually needs. In a real system, the read side often has its own storage, its own models, and its own performance constraints. A table view showing 10,000 records with sorting, filtering, and pagination is not served by loading an aggregate and calling a method — it is served by a read model optimised for that query.

**When it matters.** Any product where the read model diverges from the write model. Any product with list views, dashboards, search, or reporting. In practice, this means every product beyond a simple CRUD app.

**What happens without a solution.** Teams either query aggregates directly (which violates the aggregate's purpose as a consistency boundary and produces poor performance) or build read models ad hoc without any traceability to the specification. The view specifications from Gap 1 define what data is needed; the CQRS pattern defines how that data is served.

**v2 proposal: Read Model Derivation.**

The `view-to-query-model` skill (introduced in Gap 1) produces a read model specification for each view:

```
Read Model: RM-XX
Serves: V-XX [view spec ID]
Source events: [domain events that update this read model]
Schema:
  - [field name]: [type] — derived from [aggregate.field]
  - [field name]: [type] — computed from [event data]
Indexes: [fields that need indexes for the query patterns in the view spec]
Staleness tolerance: [real-time | eventual — N seconds | batch — N minutes]
```

Read models are projections of domain events. The domain raises events (this is already in Threadline); the read model consumes those events and maintains a denormalised view of the data. The read model specification is derived from the view spec (what data is needed) and the domain model (what events carry that data).

Integration tests for read models verify that consuming a sequence of domain events produces the expected read model state. These tests are named after the view spec they support: `RM-01 for V-01: projects TableCreated into table list view`.

---

### Gap 5: API Contract Design

**The problem.** The controller is mentioned as a driving adapter, but REST/GraphQL endpoint design, versioning, pagination, error response formats, and API documentation receive no coverage. The gap between "the interactor returns a Result" and "the API returns a well-designed response" is real and consequential. Teams need to decide on URL structures, HTTP methods, status codes, error envelope formats, pagination strategies, and content negotiation — none of which are derived from the use case.

**When it matters.** Any product with an API consumed by a frontend, a mobile app, a third-party integration, or another service.

**v2 proposal: API Contract Specification.**

Introduce an API contract artefact produced alongside the technical design in Phase 3B:

```
Endpoint: POST /forms/{formId}/submissions
Implements: UC-02 — Submit data via a custom form
Actor: End user (public)
Auth: None (public endpoint, rate-limited per QA-03)

Request:
  Path params: formId (string, required)
  Body: { [fieldId]: value, ... }

Response — success:
  201 Created → { recordId: string, status: "active" }
  Derived from: UC-02 postconditions

Response — errors:
  422 Unprocessable Entity → { errors: [{ field, code, message }] }
  Derived from: UC-02 ext 4a, 4b (ValidationError)
  
  202 Accepted → { recordId: string, status: "pending" }
  Derived from: UC-02 ext 6a (approval mode)
  
  404 Not Found → { error: "form_not_found" }
  Derived from: FormNotPublishedError

Pagination: N/A (single record creation)
Versioning: URL prefix /v1/
```

The API contract is derived from the interactor's return type (success responses), the typed domain errors (error responses), and the quality attribute scenarios (rate limiting, auth requirements). Each response explicitly references the use case element it implements, maintaining the Threadline.

The `use-case-to-technical-design` skill is updated to produce API contracts as an additional output. API contracts are reviewed at Gate 2. The acceptance tests reference the API contract for expected status codes and response shapes.

---

### Gap 6: Frontend Implementation

**The problem.** The design layer produces a Figma specification, but there is no skill for translating that into actual component code. The gap between a Figma file and a working React/Vue/Svelte application is where many teams spend the majority of their development time. The component inventory names variants after conditions, but there is no guidance on how those variants become props, state machines, or conditional rendering logic in a frontend framework.

**When it matters.** Every product with a user interface.

**v2 proposal: Frontend Component Specification.**

Introduce a `component-to-code` skill that takes the component inventory and produces framework-specific component specifications:

```
Component: FormFieldBlock
Framework: React + TypeScript

Props:
  fieldId: string
  label: string
  type: "text" | "number" | "date" | "select"
  required: boolean
  value: string | null
  error: { code: string, message: string } | null
  disabled: boolean

States (derived from component inventory variants):
  FormFieldBlock/empty       → value=null, error=null
  FormFieldBlock/filled      → value=string, error=null
  FormFieldBlock/required-error → value=null, error={ code: "required", ... }
  FormFieldBlock/type-error  → value=string, error={ code: "type_mismatch", ... }
  FormFieldBlock/disabled    → disabled=true

Derived from:
  S-02-01 (empty), S-02-02 (filled), S-02-04 (ext 4a), S-02-05 (ext 4b)
```

The component specification maintains the Threadline: each state maps to a screen ID, which maps to a use case step or extension. A developer implementing the component can trace every prop and every conditional branch back to the requirement that demanded it.

This skill is optional in the pipeline — teams with established frontend patterns may prefer to derive their own component code. But for teams using AI-assisted frontend generation, the specification serves the same role as the TDD test suite: it constrains the AI's output to match the specification.

---

## Part Three: What Will Cause Friction

These are not gaps in the methodology — they are adoption and execution challenges that teams will encounter regardless of how well the methodology is designed.

---

### Friction 1: Maintenance Burden

**The problem.** "When a use case changes, everything changes" is the right principle, but the regeneration cost is real. A single-word change to an extension in UC-02 can trigger updates to the screen inventory, the Excalidraw file, the component inventory, the Figma spec, the TDD test suite, the technical design, and the AI generation prompts. Teams under delivery pressure will be tempted to patch a test or a Figma frame directly rather than going back to the use case and cascading the change.

**What happens.** The Threadline breaks silently. The test still passes, but it no longer matches the use case. The Figma frame looks right, but it no longer matches the screen inventory. Over weeks, the specification and the implementation drift apart, and the traceability chain — the entire value proposition — degrades.

**v2 mitigation: Drift Detection.**

Introduce a `threadline-audit` skill that can be run at any time to verify the chain is intact:

```
Threadline Audit Report

UC-02: Submit data via a custom form

  Screen inventory:
    ✓ S-02-01 traces to Step 1–2
    ✓ S-02-04 traces to Ext 4a
    ✗ S-02-08 exists in inventory but has no corresponding step or extension
      → Possible drift: screen was added without updating the use case

  Test suite:
    ✓ UC-02 happy path test exists
    ✓ UC-02 ext 4a test exists
    ✗ UC-02 ext 6a: test name references ext 6a but assertion checks for "active" not "pending"
      → Possible drift: test was patched without updating from use case

  Technical design:
    ✓ Form.submit() invariant guard matches precondition
    ✗ SubmitFormUseCase interactor has no step-number comment on line 47
      → Possible drift: interactor was refactored without updating comments
```

The audit skill reads the use cases and compares them against the artefacts. It does not fix anything — it surfaces the drift so a human can decide whether to update the use case or regenerate the artefact. Running the audit before each release is a lightweight way to maintain the Threadline without regenerating everything on every change.

Additionally, introduce a **change impact report** that the pipeline orchestrator produces when a use case is modified:

```
UC-02 change detected: Extension 6a modified

Affected artefacts:
  - screen-inventory.md: S-02-06 (confirmation — pending) — REVIEW NEEDED
  - UC-02-form-submission.excalidraw: ext 6a branch — REGENERATE
  - component-inventory.md: StatusBadge/pending variant — REVIEW NEEDED
  - figma-spec.md: UC-02 page, ext 6a frames — REVIEW NEEDED
  - tests/acceptance/UC-02.test.ts: ext 6a scenario — REGENERATE
  - tests/unit/domain/Form.test.ts: ext 6a test — REGENERATE
  - technical-design/domain/Form.ts: ext 6a branch — REVIEW NEEDED
  - prompts/UC-02-implementation.md — REGENERATE
```

This makes the cost of a change visible before the team decides whether to absorb it.

---

### Friction 2: Adoption Curve

**The problem.** Threadline is a lot of process. Fully dressed format, six review dimensions, four test levels, ten skills, two gates, naming conventions for tests, screens, components, Figma pages, and code comments. A team of two shipping fast will find this crushing. A team of twenty that has never written a use case will need months to adopt.

**v2 mitigation: Threadline Lite.**

Introduce a minimum viable version of the methodology that preserves the core traceability without the full ceremony. Threadline Lite runs four skills instead of ten and skips the design pipeline entirely.

```
THREADLINE LITE

Phase 1: use-case-writer → produce use cases
Phase 2: use-case-reviewer → review (Gate 1 — can be self-review for solo devs)
Phase 3: use-case-to-tdd → produce test skeletons (all red)
Phase 4: use-case-to-prompt → produce implementation prompts referencing tests

Skip: screen-inventory, excalidraw-generator, component-inventory, figma-spec,
      use-case-to-technical-design

Gate 2 is replaced by: developer reviews test skeletons before running prompts
```

Threadline Lite is appropriate for: APIs with no UI, internal tools, backend services, solo developers, prototypes that have graduated from discovery and need production-quality code, and teams adopting Threadline incrementally.

The upgrade path from Lite to Full is additive: add the design pipeline when you have a designer. Add the technical design skill when you have a team large enough to benefit from explicit architecture documentation. The use cases and tests produced in Lite are fully compatible with the Full pipeline.

**Adoption sequence for teams new to Threadline:**

1. **Week 1–2:** Write use cases for new features only. Skip everything else. Get comfortable with the format.
2. **Week 3–4:** Add the reviewer. Run it on every use case before implementation. Learn the six dimensions.
3. **Week 5–8:** Add the TDD pipeline. Write tests from use cases before implementing. This is the hardest habit to build and the most valuable.
4. **Month 3+:** Add the design pipeline if you have a designer. Add the technical design if the team is growing. Add the full orchestrator when the skills feel natural.

---

### Friction 3: The Discovery Problem

**The problem.** Threadline is implicitly waterfall-shaped: specify everything, then design everything, then build everything. But many features need prototyping and user feedback before the spec can be written confidently. The methodology assumes you know what to build. It does not address the common situation where the goal is clear but the mechanism is not — where the team needs to build something, put it in front of users, learn from their behaviour, and then write the real spec.

**What happens.** Teams either skip Threadline for experimental features (losing all traceability) or write speculative use cases that change so frequently that the regeneration cost dominates the development time.

**v2 proposal: Phase 0 — Discovery.**

Acknowledge that the use case is written after the goal is validated, not before. Introduce an explicit discovery phase that precedes Threadline:

```
PHASE 0: DISCOVERY (before Threadline begins)

Purpose: Validate that the goal is worth building and that the mechanism is viable

Activities:
  - User interviews, market research, stakeholder alignment
  - Paper prototypes, Figma explorations, clickable mockups
  - Technical spikes — throwaway code that answers "can we build this?"
  - Competitive analysis
  
Outputs:
  - A validated goal statement (one sentence — this becomes the use case Goal)
  - A list of confirmed actors (these become the use case actors)
  - A rough flow (this becomes the basis for the main success scenario)
  - Known constraints and edge cases (these become preconditions and extensions)
  - A decision: "We are confident enough to threadline this"
  
NOT outputs:
  - Production code (spikes are thrown away)
  - Final designs (Figma explorations are not the Figma spec)
  - Architecture decisions (the architecture is derived from the use case, not from the spike)

Transition to Phase 1:
  When the team can articulate the goal in one sentence and list the actors,
  they are ready to write the use case. Discovery artefacts become inputs
  to use-case-writer, not outputs of the pipeline.
```

Phase 0 is explicitly outside the Threadline pipeline. No skills are invoked. No naming conventions apply. No traceability is expected. The purpose is to reach the point where the team is confident enough to commit to a specification — and to be honest that this confidence does not exist at the start of every feature.

The key discipline: when Phase 0 ends and Phase 1 begins, the spike code is discarded. It is not refactored into production code. The production code is generated from the Threadline pipeline, constrained by tests derived from the use case. The spike's job was to inform the use case, not to become the implementation.

---

### Friction 4: Gate Fatigue

**The problem.** Two mandatory human review gates per feature is the right model for a team shipping one or two features per sprint. But a team shipping five use cases a week will find that the gates become bottlenecks — especially Gate 2, which requires reviewing Excalidraw diagrams, technical design, test skeletons, and Figma spec simultaneously.

**v2 mitigation: Gate Scaling Policies.**

Define three gate modes based on team throughput:

**Standard mode (1–3 use cases per sprint):** Both gates are synchronous review meetings. Gate 1 involves the author and one reviewer. Gate 2 involves the author, one developer, and one designer.

**High-throughput mode (4–10 use cases per sprint):** Gate 1 is asynchronous — the reviewer posts the review and the author resolves issues without a meeting. Gate 2 is batched — review all artefacts from the sprint in one session rather than per-use-case.

**Solo mode (one developer, no designer):** Gate 1 is self-review using the six-dimension checklist (the reviewer skill runs, the developer reads the output and self-certifies). Gate 2 is replaced by a test skeleton review — the developer reads the test names and verifies they match the use case before proceeding to implementation prompts.

The gate is never skipped. What changes is who participates and whether it is synchronous.

---

### Friction 5: Excalidraw Generation Quality

**The problem.** AI-generated Excalidraw JSON is finicky. Elements overlap, arrows miss their targets, text gets clipped by container boundaries, lane backgrounds obscure step boxes, and the hand-drawn roughness sometimes produces unreadable text. In practice, generated diagrams will need manual cleanup about half the time.

**v2 mitigation: Honest quality expectations and a cleanup checklist.**

The `excalidraw-generator` skill should set expectations explicitly:

```
OUTPUT QUALITY NOTE

The generated .excalidraw file is a 70% draft. It will contain the correct
elements in approximately the correct positions, but it will likely require
manual adjustment in Excalidraw before it is ready for stakeholder review.

Common issues to fix manually:
  - Arrow endpoints not touching target boxes — drag endpoints to snap
  - Text clipped by container — resize the container or reduce font size
  - Extension boxes overlapping — increase vertical spacing
  - Lane backgrounds obscuring elements — send lane rectangles to back
  - Legend position — move to bottom-right if it overlaps content

Time to fix: 5–15 minutes per diagram. This is faster than drawing from scratch
and preserves the correct elements and structure.
```

Additionally, the skill should produce a **verification checklist** alongside the file:

```
UC-02 Excalidraw Verification:
  [ ] All 6 MSS steps present as boxes in correct lanes
  [ ] Extensions 4a, 4b, 6a present as branches below system lane
  [ ] Screen sketches present above top lane for steps 1, 3, 6
  [ ] Legend present and readable
  [ ] No overlapping elements
  [ ] Arrows connect correct source and target boxes
```

---

## Part Four: The v2 Pipeline DAG

With the additions above, the Threadline v2 pipeline DAG expands:

```
PHASE 0: DISCOVERY (outside pipeline)
  │
  │  "We are confident enough to threadline this"
  │
  ▼
INPUT: use case goals, actors, known constraints, QA requirements
  │
  ▼
[Phase 1] use-case-writer + quality-attribute-writer
  │
  ▼
[Phase 2] use-case-reviewer  ◄──── GATE 1: Human Review
  │
  │  (all use cases READY)
  │
  ├────────────────────────────────────────────────────────────────┐
  │                    │                    │                       │
  ▼                    ▼                    ▼                       ▼
[3A] use-case       [3B] use-case-to-   [3C] screen-inventory   [3D] view-spec
  -to-tdd                technical-design    │                    -writer
  │                    + API contracts       ├─► excalidraw          │
  │                    + cross-cutting       │   -generator          │
  │                      concern registry    └─► component           │
  │                    │                         -inventory          │
  │                    │                              │              │
  │                    │                         figma-spec          │
  │                    │                         + component         │
  │                    │                           -to-code          │
  │                    │                              │              │
  └────────────────────┴──────────────────────────────┴──────────────┘
                                 │
                                 ▼
                     GATE 2: Human Review
                                 │
                                 ▼
                     [Phase 5] use-case-to-prompt
                                 │
                                 ▼
                         THREADLINE COMPLETE
                                 │
                         (periodic)
                                 ▼
                         threadline-audit
```

### New Skills in v2

| Skill | Input | Output |
|---|---|---|
| `quality-attribute-writer` | Use cases + system constraints | QA scenario specifications |
| `view-spec-writer` | Product description + domain model | View specifications for read-heavy features |
| `view-to-query-model` | View specs + domain events | Read model specifications |
| `component-to-code` | Component inventory + framework | Framework-specific component specifications |
| `threadline-audit` | All artefacts | Drift detection report |
| `api-contract-writer` | Use cases + technical design | API endpoint specifications |

### Updated Skills in v2

| Skill | What changed |
|---|---|
| `use-case-to-technical-design` | Now also produces cross-cutting concern registry and API contracts |
| `screen-inventory` | Now accepts view specs as additional input alongside use cases |
| `pipeline-orchestrator` | Updated DAG with new branches, Phase 0 acknowledgement, gate scaling |
| `excalidraw-generator` | Honest quality expectations, verification checklist output |

---

## Part Five: Threadline Lite Specification

For teams that need the core value without the full ceremony.

### What's In

```
Phase 1: use-case-writer
Phase 2: use-case-reviewer (Gate 1 — can be self-review)
Phase 3: use-case-to-tdd (test skeletons, all red)
Phase 4: use-case-to-prompt (implementation prompts referencing tests)
```

### What's Out

Screen inventory, Excalidraw, component inventory, Figma spec, full technical design, API contracts, view specs, quality attribute scenarios, component-to-code.

### When to Use

- APIs with no UI
- Internal tools and admin panels
- Backend services and workers
- Solo developers
- Teams adopting Threadline incrementally (start with Lite, add skills over time)
- Prototypes that have graduated from discovery

### What You Lose

- No spatial visualisation of the flow (no Excalidraw)
- No design traceability (no Figma spec)
- No explicit architecture documentation (architecture emerges from TDD)
- No component library derivation

### What You Keep

- The use case as single source of truth
- The review gate (specification quality)
- The four-level TDD test suite (testing quality)
- The TDD-aware AI prompts (implementation quality)
- The naming conventions (traceability from test name to use case element)

### Upgrade Path

Every artefact produced by Threadline Lite is fully compatible with the Full pipeline. When the team is ready:

1. Add `screen-inventory` — takes the existing use cases as input, no changes needed
2. Add `excalidraw-generator` — takes the existing use cases + new screen inventory
3. Add `component-inventory` + `figma-spec` — takes the screen inventory
4. Add `use-case-to-technical-design` — takes the existing use cases, produces explicit architecture
5. Add the remaining v2 skills as needed

No use case needs to be rewritten. No test needs to be renamed. The thread holds.

---

## Part Six: Summary of Changes from v1 to v2

| Area | v1 | v2 |
|---|---|---|
| Read-heavy features | Not covered | View specifications + read model derivation |
| Non-functional requirements | Mentioned as "stakeholders & interests" | Quality attribute scenarios with dedicated skill |
| Cross-cutting concerns | Implicit in shared kernel | Explicit cross-cutting concern registry |
| API design | Not covered | API contract specifications derived from use cases |
| Frontend components | Figma spec only | Component-to-code skill for framework-specific specs |
| Discovery | Not covered | Phase 0 explicitly outside pipeline |
| Maintenance | "Regenerate everything" | Drift detection audit + change impact reports |
| Adoption | Full pipeline only | Threadline Lite for incremental adoption |
| Gate scaling | Two mandatory synchronous gates | Three gate modes: standard, high-throughput, solo |
| Excalidraw quality | Implied to be production-ready | Honest 70% draft expectation + verification checklist |

### What Did Not Change

The core of Threadline is unchanged in v2:

- The fully dressed use case format
- The six-dimension review
- The mechanical derivation from use case elements to architecture
- The four-level TDD test suite
- The TDD-aware AI generation prompts
- The bidirectional traceability chain
- The ten original skills (updated but not replaced)
- The eleven principles
- The two mandatory human review gates (now with scaling policies)

The foundation holds. v2 is additive, not structural.

---

*This document is the gap analysis and roadmap for Threadline v2. It should be read alongside the Threadline Master Guide, which defines the complete v1 methodology. The proposals here are intended to be implemented incrementally — each gap can be addressed independently without requiring the others.*
