# Threadline

## A Specification-Driven Methodology for Delivering Features from Use Cases to Code

---

## Preface

Threadline is a methodology for building software products in which a single specification artifact — the fully dressed use case — drives every downstream activity: product design, system architecture, test-driven development, and AI-assisted code generation. The name refers to the unbroken thread of traceability that runs from a use case element through every artifact the pipeline produces: from a precondition to an empty state screen, from an extension to a typed domain error, from a postcondition to a test assertion. When you can follow that thread in both directions — forward from spec to code, backward from bug to requirement — you have a Threadline.

Threadline is not a collection of loosely related practices. It is a coherent pipeline in which each phase receives its inputs from the phase before it, and every output remains traceable to the specification that required it. The methodology is codified as a suite of ten AI-automatable skills that can be orchestrated as an agentic pipeline, with two mandatory human review gates that cannot be automated away.

The intended audience is software teams — developers, designers, product managers, and technical leads — who want a rigorous, repeatable process that scales from a two-person startup to a multi-team organisation.

This guide is structured in six parts. Part One covers the specification layer: how to write, classify, and review use cases. Part Two covers the design layer: how to extract screens, produce Excalidraw flow diagrams, derive component libraries, and structure Figma files. Part Three covers the architecture layer: how to translate use cases into Clean Architecture with Domain-Driven Design. Part Four covers the testing layer: how to apply test-driven development at every architectural layer. Part Five covers the implementation layer: how to produce AI generation prompts constrained by pre-existing tests. Part Six covers the Threadline pipeline itself: how to orchestrate the entire process as a sequence of skills with human review gates.

---

# Part One: The Specification Layer

---

## Chapter 1: Why Use Cases

Software projects fail not because developers write bad code, but because they build the wrong thing. The root cause is almost always a weak specification — user stories that assume context, tickets that omit failure paths, PRDs that confuse mechanism with intent.

The fully dressed use case format addresses this at the source. A use case is a structured description of how an actor achieves a goal by interacting with a system. Unlike a user story, it explicitly captures every actor involved, every precondition that must hold, every step in the successful scenario, every alternative and failure path, and every observable outcome when the scenario ends. It is precise enough for a machine to reason about, and readable enough for a non-technical stakeholder to review.

This precision is what makes use cases the right foundation for Threadline. A use case is not just a specification — it is simultaneously the brief for a UI flow diagram, the source of domain invariants in the architecture, the derivation of every test case, and the prompt for AI code generation. Written once and reviewed carefully, it pays dividends across every subsequent phase of development. The thread begins here.

Each element of the use case maps directly to something every downstream consumer needs:

| Use case element | Design use | Architecture use | Test use | AI prompt use |
|---|---|---|---|---|
| Primary actor + goal | Actor interface mode | Driving port / interactor method | Unit of work scope | Generation scope |
| Preconditions | Empty state screens | Aggregate invariant guards | Given clause / test setup | Precondition enforcement |
| Main success scenario | Screen flow left to right | Interactor method body | Happy-path integration test | Implementation steps |
| Extensions | Error states, gates, variants | Typed errors, domain events | Edge case unit tests | Mandatory error handlers |
| Postconditions | Confirmation states | Return type contracts | Then clause / assertions | Return value verification |

This table is the Threadline in miniature. One element, five uses, zero information lost.

---

## Chapter 2: Goal Levels

Before writing a use case, classify the request by goal level. Goal level controls scope and prevents the most common specification failure: writing at the wrong level of abstraction.

There are three goal levels.

A **summary-level** use case describes a business capability that spans multiple user goals, such as "manage a shared workspace" or "handle billing". Summary-level use cases are too broad to implement directly. They are useful for architecture and module design decisions, but should always be decomposed into sea-level use cases before any code is written. Asking an AI to implement a summary-level use case in one shot produces plausible but wrong architecture.

A **sea-level** use case describes a single user goal achievable in one sitting: "submit data via a custom form", "invite a collaborator to a workspace", "publish a page to a public URL". Sea level is Threadline's primary working level. Every implementation task, every design flow, and every test suite corresponds to exactly one sea-level use case.

A **subfunction-level** use case describes a step within a sea-level use case: "validate a required field", "resolve a relation field". Subfunction-level use cases are useful when a complex step needs to be documented separately, but they should never be the primary specification for a feature.

The default is sea level. If a request is too broad for a single sea-level use case — if the title requires the word "and", or if the scenario spans more than one session — split it. If a request is too narrow — fewer than three meaningful steps, no preconditions worth stating — flag it as a subfunction and identify the parent use case it belongs to.

Goal levels also control design fidelity and the scope of downstream artifacts:

| Goal level | Design output | Architecture output | Test output |
|---|---|---|---|
| Summary | Information architecture, nav structure, journey map | Bounded context map, module boundaries | No tests — too broad |
| Sea level | Wireframe flow, screen by screen | Aggregates, interactors, ports, adapters | Full Threadline suite at all four levels |
| Subfunction | High-fidelity component design, interaction spec | Single method or value object | Unit test only |

---

## Chapter 3: Writing Use Cases

Every sea-level use case is written in the fully dressed format. The format has seven fields.

**Use Case ID** is a sequential identifier: UC-01, UC-02, and so on. IDs are permanent references used across every Threadline artifact — screen IDs, test names, Figma page names, interactor class names, Excalidraw file names. Never renumber existing use cases.

**Title** is a verb-noun phrase in the present tense and active voice: "Submit data via a custom form", "Create a shared data table". If the title contains "and", split the use case.

**Primary actor** is the role that initiates the goal. Named by role, not by persona: "workspace editor", not "Sarah". In automated flows the primary actor may be a system.

**Goal** is one sentence describing what the actor wants to achieve — not how. "Add a record to a shared table without accessing the table directly" is a goal. "POST to the records API endpoint" is a mechanism. Goals endure; mechanisms change.

**Preconditions** are the conditions that must be true before step one can be taken. Write them as binary assertions: either true or false. "A form linked to the target table has been published" is a precondition. "User is sort of set up in the system" is not. In Threadline, every precondition whose absence is not handled by an extension defines an empty state that the UI must design for and that the domain must guard against. Preconditions are where the thread forks into two paths: the happy path and the empty state.

**Main success scenario** is the numbered happy path. Each step has one clear subject — either the actor or the system — and describes what happens, not how. "System validates required fields" is a step. "System iterates through the fields array and calls the validate() method on each" is an implementation detail that does not belong in the specification. The main success scenario should have between four and eight steps. Fewer than four suggests a subfunction; more than eight suggests the use case should be split.

**Extensions** are the alternative and failure paths. Each extension references the step it branches from: "4a." means a branch at step 4. Multiple branches from the same step are labelled sequentially: 4a, 4b, 4c. Extensions cover validation failures, precondition violations, alternate success paths, permission boundaries, and system errors. Extensions are not optional — they are where the vast majority of bugs originate, and omitting them from the specification means those bugs will be discovered in production rather than in a test. Extensions are also the most valuable input for design: each extension becomes a screen state, an error state, a gate, or a notification that most teams would otherwise never spec. In Threadline, extensions are where the thread branches — and every branch must be followed to its end.

**Postconditions** are the observable outcomes when the scenario ends successfully. Write them from the outside: "A new record appears in the owner's table view" is observable. "The database insert completed successfully" is internal. Postconditions must be verifiable by a tester without reading the source code. They become the `Then` clauses of acceptance tests and the return type contracts of interactors.

### The Template

```
Use Case ID: UC-XX
Title: [verb + noun phrase, present tense, active voice]
Primary actor: [role name — not a persona name]
Goal: [one sentence — what the actor wants to achieve, not how]
Preconditions:
  - [Binary assertion 1]
  - [Binary assertion 2]

Main success scenario:
  1. [Declarative step — actor action or visible system response]
  2. [...]
  3. [...]

Extensions:
  Xa. [Condition at step X] → [outcome]
  Xb. [Condition at step X] → [outcome]

Postconditions:
  - [Observable outcome 1]
  - [Observable outcome 2]
```

### Quality Rules

Steps are **declarative**, not procedural. "System validates required fields" — not "System loops through fields and checks each one."

Extensions **reference the step they branch from**. "4a." means something goes wrong at step 4.

Preconditions are **binary assertions**. They define what empty states and onboarding flows need to handle.

Postconditions describe **observable outcomes**, not internal state. "A new record appears in the table view" — not "The database insert completed."

### Worked Example: UC-02 — Submit Data via a Custom Form

```
Use Case ID: UC-02
Title: Submit data via a custom form
Primary actor: End user (public or invited)
Goal: Add a record to a table without accessing the table directly
Preconditions:
  - A form linked to the target table has been published

Main success scenario:
  1. User opens the form URL
  2. System renders fields defined by the form builder
  3. User fills in fields and submits
  4. System validates required fields and field types
  5. System writes a new record to the linked table
  6. System shows a confirmation message (and optionally emails the submitter)

Extensions:
  4a. Required field is empty → system highlights the field, blocks submission
  4b. Field fails type validation (e.g. non-date in date field) → inline error, no submission
  6a. Table owner has set "require approval" → record written with status Pending,
      owner is notified

Postconditions:
  - A new record exists in the table
  - Submitter receives confirmation
  - Owner can review the record
```

This single use case, once threadlined, will generate: a screen inventory of seven screens and states, an Excalidraw swim-lane flow diagram, a set of Figma frames with variants for every extension, an aggregate with invariant guards derived from the precondition, typed domain errors for extensions 4a and 4b, a domain event for extension 6a, a four-level TDD test suite, and three AI generation prompts. One document in, eleven artifacts out, every one traceable back to the line that required it.

---

## Chapter 4: Reviewing Use Cases

A use case that passes review is ready to drive every downstream activity. A use case that fails review will produce bugs, design gaps, and incorrect code — regardless of how well the downstream activities are executed. The review gate is the most important quality control point in Threadline. It is where the thread is strongest or weakest.

Review every use case across six dimensions before proceeding.

### Dimension 1: Implementability

Ask: "Could a developer implement each step without making assumptions not stated in the use case?"

Flag steps that describe mechanisms rather than outcomes, steps that are ambiguous about who acts, and steps that combine multiple actions. "The form is submitted" — who submits it? "System calls the API" — which API, and what does it return?

### Dimension 2: Extension Completeness

For each step in the main success scenario, systematically ask: Can the actor's action fail? Can the system's response be something other than success? Is there a validation rule not yet modelled? Is there a permission boundary? Is there an alternate success path?

Flag any step that has no extensions and no plausible reason why failures are impossible. If more than 30% of steps have no extensions, the use case is not ready. In Threadline terms: the thread is missing branches, and missing branches become production bugs.

### Dimension 3: Precondition Completeness

Ask whether everything that must be true for step one to be reachable is stated. Flag implied preconditions, vague preconditions, and preconditions whose absence is handled nowhere — these all define empty states that will eventually need to be designed and built.

For each precondition whose absence is not covered by an extension, flag: "Empty state required — no design or extension handles the case where this precondition is not met."

### Dimension 4: Postcondition Observability

Ask whether each postcondition can be verified by a tester without reading source code. Flag postconditions that describe internal state, postconditions that are missing for affected parties, and postconditions that contradict extensions.

A postcondition that cannot be asserted in a test is not a postcondition — it is a comment. Rewrite it until it is observable. In Threadline, unobservable postconditions are dead ends — the thread cannot reach the test suite.

### Dimension 5: Actor Consistency

Ask whether every actor who appears in the steps is listed at the top of the use case, and whether every listed actor appears in at least one step. Flag actor names that are inconsistent across the document.

### Dimension 6: Goal Level Fit

Flag use cases that are too broad (title contains "and", more than ten steps, spans multiple sessions) or too narrow (fewer than three steps, no meaningful preconditions).

### The Review Gate

A use case that has critical issues in any dimension is marked NOT READY and must be revised before any downstream work begins. No code, design, test, or diagram is produced from a NOT READY use case. The Threadline does not begin until the use case is clean.

A use case with only minor informational issues may proceed with those issues acknowledged.

### Review Output Format

```
## Review: UC-XX — [Title]

| # | Dimension | Location | Problem | Suggested resolution |
|---|---|---|---|---|
| 1 | Extension completeness | Step 3 | No extension for network failure | Add 3a: network timeout → show retry |
| 2 | Precondition completeness | Preconditions | "User has editor role" — no extension if not | Add extension or note role guard |

### Empty states implied
[List of preconditions whose absence is not handled]

### Verdict: READY / NOT READY
```

---

# Part Two: The Design Layer

---

## Chapter 5: The Screen Inventory

The translation from use cases to UI design begins mechanically, not creatively. The creative work happens later, when a designer decides how to solve the design problems the use cases define. The mechanical work — extracting screens, states, and component variants — happens first and produces the inputs to every design tool. In Threadline, this extraction is how the thread passes from the specification layer into the design layer.

### Extraction Rules

Every step in the main success scenario that involves the user seeing something or making a decision maps to either a screen or a state change on a screen. Every extension maps to at least one additional screen or state. Every precondition whose absence is not handled by an extension defines an empty state.

**Rule 1 — Main success scenario steps.** Steps where the actor acts map to the screen the actor is looking at. Steps where the system responds visibly map to a state change on the current screen. Steps where the system acts invisibly have no UI — note as "no UI" and skip.

**Rule 2 — Extensions.** Validation failure extensions become error states on the current screen (not a new screen). Permission failure extensions become gate screens or error states. Alternate success path extensions become variant confirmation screens. Unmet precondition extensions become empty state screens or onboarding gates.

**Rule 3 — Preconditions not met.** For each precondition, generate an "empty state" entry: what the user sees when the precondition is not satisfied. These are often the most underdesigned screens in a product.

**Rule 4 — Actor interfaces.** Group screens by actor. If two actors see the same screen with different data or controls, create two separate entries with a shared component reference. Each distinct actor in your use cases is a persona with a different navigation structure — these are not themes on the same layout, they are different applications that share a data layer.

### Output Format

The screen inventory is a table with columns for Screen ID, Use Case reference, Step or Extension, Actor, Screen Name, Screen Type, and Notes. Screen IDs follow the Threadline pattern S-[UC number]-[sequence], such as S-02-01, S-02-02, S-02-03a. These IDs are how the thread is carried into the design layer — every screen traces back to the step that required it.

Screen types are: `full screen`, `state` (a variant of an existing screen), `modal`, `empty state`, `error state`, `confirmation`, `gate` (permission or onboarding barrier), and `notification` (email, push, in-app).

### Worked Example: UC-02 Screen Inventory

| Screen ID | Use case | Step / Extension | Actor | Screen name | Screen type |
|---|---|---|---|---|---|
| S-02-01 | UC-02 | Step 1–2 | End user | Form view — empty | Full screen |
| S-02-02 | UC-02 | Step 3 | End user | Form view — filled | State |
| S-02-03 | UC-02 | Step 6 | End user | Submission confirmation | Confirmation |
| S-02-04 | UC-02 | Ext 4a | End user | Form view — validation error | Error state |
| S-02-05 | UC-02 | Ext 4b | End user | Form view — type error | Error state |
| S-02-06 | UC-02 | Ext 6a | End user | Confirmation — pending | Confirmation |
| S-02-07 | UC-02 | Ext 6a | Owner | Approval notification | Notification |

This table becomes the design backlog. It is produced before any design tool is opened.

After the table, three derived outputs complete the inventory: a shared screens list (screens that serve multiple use cases — high-priority components), an empty states list (preconditions whose absence requires design), and an actor interface summary (how many screens each actor interacts with).

---

## Chapter 6: Excalidraw Flow Diagrams

Excalidraw is the right tool immediately after the screen inventory, before any high-fidelity UI decisions are made. Its hand-drawn aesthetic is intentional — a polished diagram signals finality; a rough diagram invites argument. At this stage, argument is productive. Stakeholders should feel comfortable pointing at a step and saying "that's wrong" — and the diagram should look like something that can be easily corrected.

In the Threadline pipeline, Excalidraw diagrams serve a specific role: they make the use cases spatial and discussable. Stakeholders who would not read a use case document will engage with a swim-lane diagram. Issues caught here — missing steps, wrong actor assignments, missing extensions — are far cheaper to fix than issues caught in Figma or in code.

### What Each Diagram Contains

One diagram per use case. Each diagram contains:

**Swim lanes** — one per actor. The primary actor's lane is at the top, the system lane in the middle, and any secondary actor lanes below. Lane backgrounds use consistent colours: light blue for end users, light gray for the system, light green for collaborators.

**Step boxes** — placed in the correct lane (the lane of whoever acts in that step), flowing left to right in the order of the numbered steps.

**Happy path arrows** — solid arrows connecting step boxes left to right.

**Extension branches** — hanging below the system lane as dashed arrows. Error extensions use amber fill and amber dashed arrows. Alternate success extensions use green fill and green dashed arrows.

**Rough screen sketches** — placed above the top lane with dotted reference lines connecting them to the step they represent. Enough to distinguish "this is a form" from "this is a table" from "this is a confirmation message."

**Sticky notes** — for unresolved questions, pinned directly onto the step they branch from.

### Visual Conventions

| Element | Style |
|---|---|
| Happy path arrows | Solid |
| Extension arrows | Dashed, amber |
| Alternate success arrows | Dashed, green |
| Screen reference lines | Dotted, gray |
| End user lane | Light blue background |
| System lane | Light gray background |
| Collaborator lane | Light green background |
| Extension boxes | Amber fill (error) or green fill (alt success) |

### Programmatic Generation

Excalidraw files are plain JSON with a `.excalidraw` extension. They can be generated programmatically from use case text, which means diagrams can be produced as part of the Threadline pipeline and kept in version control alongside the use cases they represent.

The top-level structure of every Excalidraw file:

```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://excalidraw.com",
  "elements": [...],
  "appState": {
    "gridSize": null,
    "viewBackgroundColor": "#ffffff"
  },
  "files": {}
}
```

All visual content lives in the `elements` array. Every element requires a unique 16-character alphanumeric ID, explicit x/y positioning, width, height, stroke and fill properties, and a `roughness` value of 1 for the hand-drawn aesthetic.

Files are named by use case ID: `UC-01-create-shared-table.excalidraw`. One file per use case. All files are kept in version control — they change when the use case changes, because in Threadline the spec is the source of truth and everything downstream follows.

---

## Chapter 7: The Component Inventory

Before opening Figma, derive the component library from the screen inventory. The component inventory answers: "What are the reusable pieces that compose all the screens, and what states does each piece need to support?"

### Step 1: Extract Raw Components

For each screen in the inventory, list every distinct UI element visible on it. Common elements: input fields, buttons, record rows, status badges, navigation items, empty state containers, confirmation banners, modal containers, permission gates, data view blocks, form field blocks.

### Step 2: Consolidate Into Named Components

Group elements that represent the same logical concept across different screens. A form field in UC-02 and a form field in UC-05 are the same component even if they have different labels.

Name components as noun phrases describing what they are, not where they appear: `FormFieldBlock` not `EventSubmissionField`. `RecordRow` not `EventTableRow`. `StatusBadge` not `ConfirmedGreenPill`.

### Step 3: Define Variants

For each component, list every state it can be in, deriving states from the screen inventory:

- **Main success scenario states:** the component in the happy path (e.g. `FormFieldBlock/empty`, `FormFieldBlock/filled`)
- **Extension states:** the component in each extension (e.g. `FormFieldBlock/required-error`, `FormFieldBlock/type-error`)
- **Permission states:** the component for restricted actors (e.g. `TableColumn/hidden`, `TableColumn/view-only`)
- **Empty states:** the component before data exists (e.g. `DataViewBlock/no-table-linked`, `RecordRow/skeleton`)

**Name variants after the condition that produces them, not the visual treatment.** This is a core Threadline principle. A variant named `required-error` is traceable to use case extension 4a. A variant named `red-outline` is not. Condition-based names survive a visual redesign; treatment-based names do not. The thread must survive the redesign.

---

## Chapter 8: The Figma Specification

Figma enters once Excalidraw sketches have been reviewed and the screen inventory is agreed. The critical move is to bring the Threadline structure into Figma, not leave it behind.

### File Structure

The Figma file is organised around use cases:

1. `_Cover` — project name, use case suite name, last updated date, owner
2. `_Components` — the full component library from the component inventory
3. One page per use case, named by ID and title: `UC-01 · Create shared table`
4. `_Archive` — deprecated screens moved here rather than deleted

Pages prefixed with `_` sort to the top. Use case pages are ordered by dependency. Never put screens from multiple use cases on the same page — it breaks the thread.

### Screen Layout Within Each Page

Within each use case page, screens are laid out left to right in scenario step order — the same sequence as the numbered steps. Extension screens are placed below the happy path, visually offset, directly below the step they branch from. This spatial layout is the documentation: anyone opening the file can read the design the same way they read the use case. The layout *is* the Threadline made visible.

### Component Naming

Map the component inventory to Figma components. Name component variants after the condition that produces them:

- `FormField/required-empty` not `FormField/red-outline`
- `RecordRow/restricted` not `RecordRow/grayed-out`
- `TableColumn/hidden` not `TableColumn/dimmed`

### Prototype Flows

Figma's prototype flows mirror use case sequences: one flow per main success scenario (tracing steps left to right), and one flow per extension (branching off the step it modifies, labelled by extension ID).

### Annotations

Use an annotation layer — a separate locked frame overlaid on the design — that references the use case step driving each screen:

```
UC-02 · step 4 · validation state
UC-02 · ext 4a · required field empty
```

Annotations are how the thread is visible in the design file. They survive screen redesigns because they live on a separate locked layer.

### The Threadline Handoff

A developer receiving this work gets three things that all point at the same truth:

1. The use case document (the spec)
2. The Excalidraw flow (the navigation model)
3. The Figma file (the visual implementation)

Each one can be read independently, and each one references the same step and extension numbers. That shared reference system is the Threadline.

---

# Part Three: The Architecture Layer

---

## Chapter 9: From Use Cases to Clean Architecture

The architecture derived from use cases is a Clean Architecture with Domain-Driven Design filling the domain layer. Clean Architecture provides the layer structure and the dependency rule. DDD provides the vocabulary and design patterns for what lives inside the domain layer.

In Threadline, the derivation is deterministic. Given a well-written use case, the architecture is not a creative exercise — it is a translation. The thread passes from the specification layer through the architecture layer without losing information.

### The Four Layers

```
┌────────────────────────────────────────────┐
│  Infrastructure (Frameworks & Drivers)     │  HTTP, DB, queues, email
│  ┌──────────────────────────────────────┐  │
│  │  Adapters (Interface Adapters)       │  │  Controllers, repos, presenters
│  │  ┌────────────────────────────────┐  │  │
│  │  │  Application (Use Cases)       │  │  │  Interactors, event handlers
│  │  │  ┌──────────────────────────┐  │  │  │
│  │  │  │  Domain (Entities)       │  │  │  │  Aggregates, value objects,
│  │  │  │                          │  │  │  │  domain events, typed errors
│  │  │  └──────────────────────────┘  │  │  │
│  │  └────────────────────────────────┘  │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

### The Core Mapping

| Use case element | Architectural concept |
|---|---|
| Primary actor | Driving port — the primary adapter calls the use case |
| Goal | Public method on the use case interactor |
| Preconditions | Invariant guards inside aggregate methods |
| Main success scenario steps | The interactor method body |
| Extensions — validation failures | Typed domain errors returned from aggregate methods |
| Extensions — alternate success paths | Domain events raised by the aggregate |
| Postconditions | Return type contract + integration test assertions |
| Stakeholders & interests | Non-functional requirements on the adapter layer |

There is no invention required — only translation. The thread passes through unchanged.

---

## Chapter 10: Bounded Contexts

Before designing any aggregate, identify the bounded contexts. A bounded context is a semantic boundary inside which a particular domain model is defined and consistent.

### Rules for Grouping

Group use cases into contexts by asking what changes together and what speaks the same language:

- Use cases sharing the same primary aggregate belong in the same context
- Use cases with different primary actors almost always belong in different contexts
- If two use cases share a noun but mean different things by it, that noun lives in both contexts under different names — do not unify them
- A shared kernel is a small set of types shared across contexts without duplication (e.g. UserId, WorkspaceId)

Bounded contexts communicate through domain events, never by sharing aggregates or importing each other's domain types directly.

### Worked Example: Five Use Cases, Four Contexts

```
┌──────────────────────────────────────────────┐
│  Workspace Context                            │
│  UC-01 (create table), UC-04 (field access)  │
│  Aggregates: Workspace, Table, Field         │
│  Language: schema, field, permission, role    │
└──────────────────────────────────────────────┘
         │ publishes TableCreated, FieldAccessChanged
         ▼
┌──────────────────────────────────────────────┐
│  Submission Context                           │
│  UC-02 (form submission)                     │
│  Aggregates: Form, Record, SubmissionPolicy  │
│  Language: submission, validation, approval   │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│  Publishing Context                           │
│  UC-03 (build page), UC-05 (fork template)   │
│  Aggregates: Page, Block, Template           │
│  Language: page, block, slug, publish, fork  │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│  Identity & Access — Shared Kernel           │
│  All use cases                               │
│  Types: UserId, WorkspaceId, Role            │
└──────────────────────────────────────────────┘
```

---

## Chapter 11: The Domain Layer

The domain layer contains aggregates, entities, value objects, and domain events. It imports nothing from outside itself.

### Value Objects

Value objects are immutable types defined by their value. Every significant domain noun should be a value object rather than a primitive. A `FormId` that validates its own contents is more expressive and safer than a `string`.

**TypeScript:**
```typescript
export class FormId {
  private constructor(public readonly value: string) {}

  static create(value: string): FormId {
    if (!value || value.trim().length === 0) {
      throw new Error('FormId cannot be empty')
    }
    return new FormId(value)
  }

  static generate(): FormId {
    return new FormId(crypto.randomUUID())
  }

  equals(other: FormId): boolean {
    return this.value === other.value
  }
}
```

**GoLang:**
```go
type FormID string

func NewFormID(value string) (FormID, error) {
    if strings.TrimSpace(value) == "" {
        return "", errors.New("FormID cannot be empty")
    }
    return FormID(value), nil
}

func GenerateFormID() FormID {
    return FormID(uuid.New().String())
}
```

### Aggregates and Invariants

An aggregate is a consistency boundary. Everything inside it changes together in one transaction. The aggregate root is the only entry point from outside. Aggregates reference each other by ID only, never by object reference.

The critical Threadline move is mapping preconditions to aggregate invariants. A precondition states what must be true before a scenario begins. The aggregate method that implements that scenario must enforce this precondition as an invariant guard — at the top of the method, before any behaviour executes. This enforcement belongs in the domain, not in controllers, not in interactors. The thread from precondition to invariant guard is direct and unbroken.

**TypeScript — Form aggregate with precondition guard and extension handling:**
```typescript
submit(values: FieldValues): Result<Record, SubmissionError> {
  // UC-02 precondition: form must be published
  if (!this.status.isPublished()) {
    return err(new FormNotPublishedError(this.id))
  }

  // UC-02 ext 4a, 4b: validate all fields
  const validation = this.validateFields(values)
  if (validation.hasErrors()) {
    return err(new ValidationError(validation.errors))
  }

  // UC-02 ext 6a: approval policy determines record status
  const record = Record.create(this.tableId, values, this.submissionPolicy)
  return ok(record)
}
```

**GoLang:**
```go
func (f *Form) Submit(values FieldValues) (*Record, error) {
  if !f.status.IsPublished() {
    return nil, &FormNotPublishedError{FormID: f.id}
  }

  if errs := f.validateFields(values); len(errs) > 0 {
    return nil, &ValidationError{Errors: errs}
  }

  return NewRecord(f.tableID, values, f.submissionPolicy), nil
}
```

Notice the comments: `// UC-02 precondition`, `// UC-02 ext 4a, 4b`, `// UC-02 ext 6a`. Those comments are the thread. They connect the code to the specification permanently.

### Typed Errors and Domain Events

Extensions split into two architectural concepts:

**Error extensions** (validation failures, precondition violations) become typed domain errors returned from aggregate methods. Each extension gets its own error type. Generic errors are never used for domain conditions. The error type carries enough information for the caller to handle the condition specifically. In Threadline terms: a `ValidationError` carries the list of field errors, which traces back to extension 4a, which traces back to the screen inventory entry S-02-04, which traces back to the Figma error state frame.

**Alternate success extensions** (UC-02 ext 6a — approval mode) become domain events raised by the aggregate. The event carries the information needed for its handler. The handler is wired in the application layer, not in the domain.

---

## Chapter 12: The Application Layer

The application layer contains use case interactors and domain event handlers. It imports from the domain layer and from port interfaces it defines itself. It imports nothing from adapters or infrastructure.

An interactor is a direct translation of the main success scenario. Each numbered step becomes one line or block. Step numbers appear as comments, maintaining the permanent Threadline traceability link.

### Rules

- One interactor class or struct per use case
- One public method: `execute(command)` returning `Result` or `error`
- The interactor **orchestrates** — it calls domain methods and repositories. It contains no business logic.
- No imports from adapters, HTTP packages, or database packages
- Every typed error from the domain is propagated unchanged — never swallowed

**TypeScript:**
```typescript
async execute(cmd: SubmitFormCommand): Promise<Result<SubmitFormResult, SubmissionError>> {
  // Steps 1–2: actor opens form, system renders — handled by adapter/query

  // Step 3: actor submits — we receive values
  const form = await this.forms.findById(FormId.create(cmd.formId))
  if (!form) return err(new FormNotPublishedError(FormId.create(cmd.formId)))

  // Step 4: validate + Step 5: write record (domain handles both)
  const result = form.submit(FieldValues.from(cmd.values))
  if (result.isErr()) return result

  // Step 5 continued: persist
  await this.records.save(result.value)

  // Step 6: publish events (handles ext 6a via event handler)
  await this.events.publishAll(result.value.domainEvents())

  return ok({
    recordId: result.value.id.value,
    status: result.value.status.value,
  })
}
```

### Ports

Ports are interfaces defined in the application layer and implemented in the adapter layer. Every external dependency — database, email service, message queue, clock, identifier generator — is a port. Port method signatures use domain types, not primitives.

### The Dependency Rule

One rule governs the entire architecture: source code dependencies point only inward.

```
infrastructure  →  adapters  →  application  →  domain
                               (ports defined      (imports
                                here, impls         nothing)
                                in adapters)
```

The dependency rule is non-negotiable. A violation is always fixable by extracting a port interface.

| Violation | Fix |
|---|---|
| Domain imports a database driver | Extract a repository port in the application layer |
| Application layer imports an HTTP package | The HTTP code belongs in an adapter |
| Application layer imports a concrete repository | Inject as the port interface it implements |
| Domain imports a clock or ID generator | Extract a port; inject it |

---

# Part Four: The Testing Layer

---

## Chapter 13: Test-Driven Development from Use Cases

Test-driven development in Threadline is not simply "write tests before code." It is a structured discipline in which every test is derived from a specific element of the use case, written at the appropriate architectural layer, and run before any implementation exists.

The result is a test suite that is permanently traceable to the specification — the thread running from use case to test name to assertion. When a test fails, the use case element that required it is immediately identifiable. When a use case changes, the tests that must change are immediately identifiable.

### The Pipeline Order Changes

Without TDD: spec → design → implement → test. With Threadline:

```
spec → review → test (red) → implement (green) → refactor → repeat
```

Tests are written immediately after use cases are reviewed, before any implementation or detailed technical design. The tests define the target; the implementation is what satisfies the target.

---

## Chapter 14: The Double Loop

The double loop is the structural model for TDD in a layered architecture.

The **outer loop** is a BDD acceptance test covering the complete user journey. It is written first and stays red until every layer is implemented. It defines "done" for the entire use case.

The **inner loop** contains unit and integration tests, one per layer, each written before the implementation it covers. The inner loops collectively make the outer loop go green.

```
┌──────────────────────────────────────────────────────┐
│  OUTER LOOP (BDD acceptance test)                    │
│                                                      │
│  Given [preconditions]                               │
│  When  [triggering step]                             │
│  Then  [postconditions]                              │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  INNER LOOP 1: Domain unit tests             │   │
│  │  Derived from: extensions + invariants       │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  INNER LOOP 2: Interactor tests              │   │
│  │  Derived from: main success scenario steps   │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  INNER LOOP 3: Integration/contract tests    │   │
│  │  Derived from: postconditions                │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### The Two TDD Schools, Reconciled

**Outside-In (London school):** Start at the acceptance test. Work inward. Mock everything at the inner boundary. Discover the API of each layer by asking "what does the layer above need from me?"

**Inside-Out (Detroit/Chicago school):** Start at the domain core. Build outward with real objects. Discover the domain model by working with it directly.

Threadline uses both in sequence:

1. Write the acceptance test first (outside-in — defines the target)
2. Drop to the domain and use inside-out TDD to build aggregates
3. Rise to the interactor using outside-in with mocked ports
4. Rise to the adapter layer using integration tests
5. The acceptance test goes green — the use case is complete

This sequence follows the dependency rule: you always implement from the inside out, but you define the target from the outside in.

---

## Chapter 15: The Four Test Levels

### Mapping Use Case Elements to Tests

| Use case element | Test level | Test construct |
|---|---|---|
| Full scenario | Acceptance | Complete BDD scenario |
| Preconditions | All levels | `Given` clause / `beforeEach` setup |
| Main success scenario | Interactor test | Mock call sequence verification |
| Extensions — validation | Domain unit test | Aggregate returns typed error |
| Extensions — alternate success | Integration test | Domain event raised, handler executed |
| Postconditions | Acceptance + integration | `Then` clause / final assertions |

### Level 1: Acceptance Tests

Acceptance tests cover the complete user journey. They are derived directly from the use case by mapping Gherkin clauses: `Given` from preconditions, `When` from the triggering step, `Then` from postconditions.

One scenario per main success scenario. One scenario per extension.

```gherkin
Feature: UC-02 Submit data via a custom form

  Scenario: UC-02 happy path — active record on valid submission
    Given a published form linked to the "Events" table
    When a public user submits with valid values
    Then the response status is 201
    And a new record exists with status "active"

  Scenario: UC-02 ext 4a — required field empty blocks submission
    Given a published form with a required "name" field
    When a user submits without a "name" value
    Then the response status is 422
    And no record is created

  Scenario: UC-02 ext 6a — approval mode writes pending status
    Given a published form with approval required
    When a user submits with valid values
    Then the response status is 202
    And a new record exists with status "pending"
    And the owner receives a notification
```

Acceptance tests exercise the full stack — HTTP request in, database state verified out — and use test doubles only for external services that cannot be run in a test environment.

### Level 2: Domain Unit Tests

Domain unit tests cover aggregates and value objects. They are pure unit tests with no infrastructure — no database, no HTTP, no I/O — and must run in milliseconds. Each extension becomes at least one domain unit test. Each invariant becomes at least one test verifying it is enforced.

These tests use the inside-out approach: they exercise real domain objects, not mocks.

### Level 3: Interactor Tests

Interactor tests verify that the use case orchestrates the domain and ports correctly. All ports are mocked. The test verifies the call sequence matches the main success scenario steps and that typed errors from the domain propagate upward unchanged.

### Level 4: Integration / Contract Tests

Integration tests verify that adapter implementations fulfil the port contracts they implement, against real infrastructure. They test one adapter at a time — not the full stack.

### Threadline Test Naming Convention

Every test name references the use case element it covers. This is how the thread reaches the test suite. A developer seeing a failing test can trace it back to the exact use case element without reading the implementation.

```
Acceptance:   UC-02: creates an active record on valid submission
Extension:    UC-02 ext 4a: required field empty blocks submission
Domain unit:  Form.submit() — required field missing returns ValidationError
Interactor:   SubmitFormUseCase — ValidationError from domain is propagated
Integration:  PostgresFormRepository.findById — returns null when not found
```

### Test Helper Conventions

Four helper types serve different purposes:

**Builders** create domain objects in specific states using the real database, for acceptance and integration tests. Fluent interface: `FormBuilder.publishedWithApproval(db)`.

**Fixtures** are in-memory aggregate instances for domain unit tests. No I/O: `publishedFormData({ requiresApproval: true })`.

**Spies** are port implementations that record calls: `expect(notifierSpy.calls).toHaveLength(1)`.

**Mocks** are port implementations that return configured values: `forms.willReturn(publishedFormFixture())`.

All helpers are named after the domain concept they support, not after the test that uses them.

---

# Part Five: The Implementation Layer

---

## Chapter 16: AI Generation Prompts

The implementation prompts produced at the end of the Threadline pipeline differ from conventional code generation prompts in one critical way: they reference pre-existing test files and instruct the AI to make those tests pass, not to generate its own tests.

This constraint closes a critical gap in AI-assisted development. Without pre-existing tests, the AI generates code and tests together, optimising for tests that pass rather than for tests that correctly specify behaviour. With pre-existing tests derived from use cases, the AI's output is constrained by the specification. The tests were derived from the use cases; the code must satisfy the tests; the specification is therefore enforced at the code level. The thread runs all the way from the use case to the passing test.

### Three Prompt Types Per Use Case

**Prompt Type 1: Data Model Prompt** — establishes the schema. Run first. Includes the preconditions and postconditions that define what entities exist and what fields they have.

**Prompt Type 2: Implementation Prompt** — references the data model and the test file names. Instructs the AI to implement in this order: value objects, domain error types, domain events, aggregate, use case interactor, port interfaces. Adapters are a separate prompt.

```
Implement [UseCase] such that all tests in [test file paths] pass.
Do not modify any test file.
Do not add any tests.
Make the existing tests pass by implementing the production code.

Implement in this order:
1. Value objects
2. Domain error types
3. Domain events
4. Aggregate
5. Use case interactor
6. Port interfaces
Stop after step 6. Adapters are a separate prompt.
```

**Prompt Type 3: Test Completion Prompt** — fills in any test helpers, builders, fixtures, and spies needed by the test skeletons.

### Prompt Quality Rules

- Insert the full use case text verbatim — do not summarise
- Extensions are mandatory — they are where 80% of real bugs live
- Every test name must reference the use case step or extension it covers
- Postconditions become the test assertions, not supplementary documentation
- If the use case references other use cases, note those dependencies
- If something is ambiguous, the AI should output a TODO comment, not invent behaviour

### Common Failure Modes

**Scope creep at generation time.** The AI invents subfunction behaviour not in the use case. Preconditions act as guardrails if enforced explicitly.

**Losing traceability.** If the AI refactors heavily, step-to-test mapping breaks. Keep test names anchored to use case IDs. A break in the naming convention is a break in the Threadline.

**Summary-level hallucination.** Never ask an AI to implement a summary-level use case in one shot.

**Skipping extensions.** Make extensions non-negotiable in the prompt template.

---

# Part Six: The Threadline Pipeline

---

## Chapter 17: The Ten Skills

The complete Threadline methodology is codified as a pipeline of ten skills, each responsible for one phase of the process. A skill is a structured document (SKILL.md) that instructs an AI agent how to perform a specific task — what inputs to expect, what to produce, what quality rules to apply, and what failure modes to handle. Skills are read by the agent at the point of invocation, not loaded into context at startup. This keeps each invocation focused and prevents context dilution across a long pipeline run.

| # | Skill | Input | Output |
|---|---|---|---|
| 1 | `use-case-writer` | Product description | Fully dressed use cases |
| 2 | `use-case-reviewer` | Use cases | Structured review, READY/NOT READY |
| 3 | `use-case-to-tdd` | Reviewed use cases | Four-level TDD test suite (all red) |
| 4 | `use-case-to-technical-design` | Reviewed use cases | Bounded contexts, aggregates, interactors, ports, adapters |
| 5 | `screen-inventory` | Reviewed use cases | Screen inventory table |
| 6 | `excalidraw-generator` | Use cases + screen inventory | One `.excalidraw` file per use case |
| 7 | `component-inventory` | Screen inventory | Component library with variants |
| 8 | `figma-spec` | Screen + component inventory | Figma file specification |
| 9 | `use-case-to-prompt` | Reviewed use cases + test file names | Three AI prompts per use case |
| 10 | `pipeline-orchestrator` | Product description + framework context | Coordinates all nine skills |

The pipeline orchestrator is the meta-skill. It reads and invokes all other skills in the correct sequence, manages the human review gates, and produces the artifact manifest at the end.

---

## Chapter 18: The Pipeline DAG

The Threadline pipeline is a directed acyclic graph with two sequential phases, three parallel branches in the middle, and two mandatory human review gates.

```
INPUT: product description / feature brief
  │
  ▼
[Phase 1] use-case-writer
  │
  ▼
[Phase 2] use-case-reviewer  ◄──── GATE 1: Human Review
  │                                 Fix issues → back to Phase 1
  │  (all use cases READY)
  │
  ├───────────────────────────────────────────────────┐
  │                              │                     │
  ▼                              ▼                     ▼
[3A] use-case-to-tdd    [3B] use-case-to-         [3C] screen-inventory
  │  (acceptance tests,       technical-design         │
  │   domain unit tests,      (bounded contexts,       ├─► excalidraw-generator
  │   interactor tests,        aggregates, ports,      │
  │   integration tests)       adapters, directory     └─► component-inventory
  │                            structure)                       │
  │                              │                              ▼
  │                              │                         figma-spec
  │                              │                              │
  └──────────────────────────────┴──────────────────────────────┘
                                 │
                                 ▼
                     GATE 2: Human Review
                     (Excalidraw + technical design +
                      TDD skeletons + Figma spec)
                                 │
                                 │ (approved)
                                 ▼
                     [Phase 5] use-case-to-prompt
                     (implementation, test, data model prompts)
                                 │
                                 ▼
                         THREADLINE COMPLETE
```

---

## Chapter 19: The Two Human Review Gates

Both gates are mandatory. They cannot be automated away. They are where human judgement enters the Threadline — and they are the reason the output can be trusted.

### Gate 1: After the Reviewer

Gate 1 enforces specification quality. The reviewer flags issues, but a human decides whether they are blocking. Some issues are structural (wrong goal level, missing extensions) and must be fixed. Others are informational (an empty state exists but the design will handle it) and can be acknowledged.

The agent cannot make this judgement — it does not know what design decisions the team has already made.

**Gate behaviour:** If ALL use cases are READY, proceed automatically. If ANY use case is NOT READY, STOP. Present the review. Wait for instruction: fix inline or return to Phase 1 with the review as context. Do not proceed without explicit confirmation. A NOT READY use case that slips past this gate contaminates every downstream artifact.

### Gate 2: After the Parallel Branches

Gate 2 enforces design and architecture quality. At this gate, a human has for the first time a spatial representation of the system (Excalidraw), a component model (Figma spec), a technical architecture (bounded contexts, aggregates, interactors), and a test suite. Reading these four artifacts together surfaces inconsistencies that would not be visible in any single artifact alone.

A screen in the inventory with no corresponding step indicates a missing step. An aggregate invariant that does not correspond to any precondition indicates an invented constraint. A test with no corresponding extension indicates a test that will never fail for the right reason. These cross-artifact inconsistencies are breaks in the Threadline — and Gate 2 is where they are caught.

This gate should involve at minimum: one person who wrote the use cases, and one person who will implement the feature.

---

## Chapter 20: Running the Threadline

### Option 1: Step-by-Step Invocation

Invoke each skill individually with a prompt pattern:

```
Read the [skill-name] SKILL.md then [action] for the following:

[input content]

[specific output instructions]
```

Check the output of each skill against its quality criteria before passing it to the next skill.

### Option 2: Single-Prompt Pipeline Initiation

If the agent supports multi-step agentic execution, initiate the entire Threadline with one prompt:

```
Read skills/pipeline-orchestrator/SKILL.md and run the complete
Threadline pipeline.

Product description:
[description]

Language(s): [TypeScript | GoLang | both]

Framework context:
[language, HTTP framework, ORM, test library, existing patterns]

Available skill files:
  skills/use-case-writer/SKILL.md
  skills/use-case-reviewer/SKILL.md
  skills/use-case-to-tdd/SKILL.md
  skills/use-case-to-technical-design/SKILL.md
  skills/screen-inventory/SKILL.md
  skills/excalidraw-generator/SKILL.md
  skills/component-inventory/SKILL.md
  skills/figma-spec/SKILL.md
  skills/use-case-to-prompt/SKILL.md

Pause at both human review gates. Do not proceed past either gate
without explicit confirmation.
```

### Option 3: Partial Runs

The pipeline supports running subsets:

- "Just write the use cases" → Phase 1 only
- "Just the tests" → Phase 1 → 2 → 3A only
- "Just the design artifacts" → Phase 1 → 2 → 3C only
- "Just the technical design" → Phase 1 → 2 → 3B only
- "Skip to the prompts" → requires Gate 2 cleared; refuse and explain why if not

The gates are not optional regardless of which branches are run. The Threadline does not have shortcuts.

---

## Chapter 21: Iterating After the Threadline

When requirements change after the pipeline has run, do not re-run the full Threadline. Target the affected use case only:

1. Edit the use case
2. Re-run `use-case-reviewer` on the changed use case
3. Update the screen inventory for the affected use case only
4. Regenerate the Excalidraw file for the affected use case
5. Update the component inventory if new variants are required
6. Update the Figma spec for the affected use case page
7. Regenerate the TDD test suite for the affected use case
8. Regenerate the implementation and test prompts for the affected use case

Do not regenerate unaffected use cases — their Screen IDs, component variants, tests, and prompts remain valid. The Threadline is scoped to the use case: when one thread changes, the others hold.

---

## Chapter 22: Failure Modes and Recovery

**use-case-writer produces a single giant use case.** The agent bundled multiple goals into one. Return with: "Split this into separate use cases — one per distinct user goal. Each title should be achievable in one sitting."

**use-case-reviewer finds no issues on a complex use case.** The agent is being too lenient. Return with: "Re-review and specifically ask: for every step, what can go wrong? For every precondition, what happens if it is not met?"

**excalidraw-generator produces a file that doesn't open.** The JSON is malformed. Return with: "Validate the JSON before writing. Check unique IDs, arrow points arrays, and top-level structure."

**component-inventory uses treatment-based variant names.** Return with: "Rename all variants to describe the condition, not the visual treatment. 'required-error' not 'red-outline'. The Threadline requires condition-based names."

**use-case-to-prompt generates tests without step references.** Return with: "Every test name must follow the pattern test_uc[ID]_step[N]_[description] or test_uc[ID]_ext[Na]_[description]. Test names without step references break the Threadline."

**Parallel branches produce conflicting outputs.** The screen inventory and technical design must agree on aggregate names and context boundaries. Surface the conflict and ask the user to decide.

**Gate 2 reveals use case gaps.** If a screen exists in the inventory that has no corresponding step in the use case, that step is missing. Return to Phase 1 — do not patch the screen inventory to paper over the gap. In Threadline, gaps are always fixed at the source.

---

## Chapter 23: The Threadline

The defining property of this methodology is bidirectional traceability — the unbroken thread from which it takes its name. Every artifact produced by the pipeline can be traced back to the use case element that required it, and every use case element can be traced forward to the artifacts that implement it.

**Following the thread forward:** A use case extension → a named test case → a typed domain error → a branch in the aggregate method → a specific HTTP status code in the controller → a specific assertion in the acceptance test → a specific error state in the Figma file → a specific variant in the component inventory → a specific branch in the Excalidraw diagram.

**Following the thread backward:** A failing test whose name contains `ext_4a` immediately identifies the use case and extension that required the behaviour. A bug in a controller response can be traced to the extension that defined it, to the use case, and to the stakeholder concern that motivated the requirement. A Figma frame annotated `UC-02 · ext 4a` traces directly to the spec.

Any break in the thread — a test named after a variable, a screen without an annotation, a component variant named after a colour, an interactor without step-number comments — is a gap in traceability and should be treated as a process defect. The thread is the value of the process. Protect it.

---

# Appendices

---

## Appendix A: The Fully Dressed Use Case Template

```
Use Case ID: UC-XX
Title: [verb + noun phrase, present tense, active voice]
Primary actor: [role name — not a persona name]
Goal: [one sentence — what the actor wants to achieve, not how]
Preconditions:
  - [Binary assertion 1]
  - [Binary assertion 2]

Main success scenario:
  1. [Declarative step — actor action or visible system response]
  2. [...]
  3. [...]

Extensions:
  Xa. [Condition at step X] → [outcome]
  Xb. [Condition at step X] → [outcome]

Postconditions:
  - [Observable outcome 1]
  - [Observable outcome 2]
```

---

## Appendix B: Threadline Review Checklist

Before marking a use case ready to threadline, verify each item.

**Title:** verb-noun phrase, present tense, no "and".

**Goal:** describes intent, not mechanism; one sentence.

**Preconditions:** each is a binary assertion; each whose absence is unhandled implies an empty state.

**Steps:** four to eight; each has one subject; declarative not procedural; no step mixes multiple actions.

**Extensions:** every step that can fail has at least one; every extension references its step number; every extension outcome is stated.

**Postconditions:** all observable without reading source; none describe internal state; none contradict extensions.

**Goal level:** achievable in one sitting; title does not contain "and"; four to eight steps.

---

## Appendix C: Threadline Naming Conventions

### Test Names

```
Acceptance:   [UC-ID] [goal summary]
              UC-02: creates an active record on valid submission

Extension:    [UC-ID] ext [Na]: [condition and outcome]
              UC-02 ext 4a: required field empty blocks submission

Domain unit:  [Aggregate].[method]() — [condition]
              Form.submit() — required field missing returns ValidationError

Interactor:   [UseCase] — [condition and call sequence note]
              SubmitFormUseCase — ValidationError from domain is propagated

Integration:  [Adapter].[method] — [condition]
              PostgresFormRepository.findById — returns null when not found
```

### Screen IDs

```
S-[UC number]-[sequence]
S-02-01   (first screen for UC-02)
S-02-04   (extension screen for UC-02 ext 4a)
```

### Component Variant Names

```
[ComponentName]/[condition-that-produces-it]
FormFieldBlock/required-error       (from UC-02 ext 4a)
FormFieldBlock/type-error           (from UC-02 ext 4b)
StatusBadge/pending                 (from UC-02 ext 6a)
```

### Figma Page Names

```
UC-XX · [Title]
UC-02 · Submit data via a custom form
```

### Figma Annotations

```
UC-XX · step N · [description]
UC-02 · step 4 · validation state
UC-02 · ext 4a · required field empty
```

### Code Comments

```
// UC-02 precondition: form must be published
// UC-02 ext 4a, 4b: validate all fields
// UC-02 ext 6a: approval policy determines status
```

---

## Appendix D: The Dependency Rule at a Glance

```
infrastructure  →  adapters  →  application  →  domain
                               (ports defined      (imports
                                here, impls         nothing)
                                in adapters)
```

| Violation | Fix |
|---|---|
| Domain imports a database driver | Extract a repository port in the application layer |
| Application layer imports an HTTP package | The HTTP code belongs in an adapter |
| Application layer imports a concrete repository | Inject as the port interface |
| Domain imports a clock or ID generator | Extract a port; inject it |

---

## Appendix E: Threadline Artifact Manifest

The complete pipeline produces this file tree:

```
outputs/
├── use-cases.md
├── tests/
│   ├── acceptance/        UC-XX-[slug].acceptance.test.[ts|go]
│   ├── unit/
│   │   ├── domain/        [Context]/[Aggregate].test.[ts|go]
│   │   └── application/   [Context]/[UseCase]UseCase.test.[ts|go]
│   └── integration/
│       └── adapters/      [Adapter]/[Repository].test.[ts|go]
├── technical-design/
│   ├── bounded-contexts.md
│   ├── domain/            [context]/[Aggregate].[ts|go]
│   ├── application/       [context]/[UseCase]UseCase.[ts|go]
│   ├── application/       [context]/ports.[ts|go]
│   ├── adapters/http/     [UseCase]Controller.[ts|go]
│   ├── adapters/[db]/     [Aggregate]Repository.[ts|go]
│   └── directory-structure.md
├── design/
│   ├── screen-inventory.md
│   ├── UC-XX-[slug].excalidraw
│   ├── component-inventory.md
│   └── figma-spec.md
└── prompts/
    ├── UC-XX-data-model.md
    ├── UC-XX-implementation.md
    └── UC-XX-tests.md
```

---

## Appendix F: Threadline Skill Suite Reference

| Skill | Trigger phrases | SKILL.md location |
|---|---|---|
| `use-case-writer` | "write a use case", "spec this feature", "define requirements" | skills/use-case-writer/SKILL.md |
| `use-case-reviewer` | "review this use case", "check this spec", "find the gaps" | skills/use-case-reviewer/SKILL.md |
| `use-case-to-tdd` | "write the tests first", "generate TDD tests", "BDD from use cases" | skills/use-case-to-tdd/SKILL.md |
| `use-case-to-technical-design` | "design the architecture", "derive aggregates", "map to DDD" | skills/use-case-to-technical-design/SKILL.md |
| `screen-inventory` | "what screens do I need", "extract the screens", "design backlog" | skills/screen-inventory/SKILL.md |
| `excalidraw-generator` | "generate excalidraw", "make a flow diagram", "visualise this" | skills/excalidraw-generator/SKILL.md |
| `component-inventory` | "what components do I need", "derive the component library" | skills/component-inventory/SKILL.md |
| `figma-spec` | "write a Figma spec", "structure the Figma file" | skills/figma-spec/SKILL.md |
| `use-case-to-prompt` | "generate code from this", "turn this into a prompt" | skills/use-case-to-prompt/SKILL.md |
| `pipeline-orchestrator` | "run the full pipeline", "threadline this feature" | skills/pipeline-orchestrator/SKILL.md |

---

## Appendix G: The Threadline Principles

These eleven principles distil the methodology. They are not rules to be followed mechanically but commitments that, taken together, produce the traceability the name promises.

**1. The use case is the specification.** Not the ticket. Not the story. Not the PRD. The fully dressed use case is the only artifact from which code, tests, and designs are derived.

**2. Extensions are not optional.** Every extension is a code path, a test case, and a design state. Omitting extensions means discovering those paths in production.

**3. Preconditions define empty states.** Every precondition that might not be met requires an empty state design and a test setup.

**4. Postconditions are executable.** A postcondition that cannot be asserted in a test is not a postcondition — it is a comment.

**5. Invariants belong in the domain.** Precondition enforcement does not belong in controllers or interactors. It belongs in the aggregate method that requires it.

**6. Extensions become typed errors or domain events, never generic exceptions.** Each failure path gets its own error type. Each alternate success path gets a domain event.

**7. The interactor orchestrates; it does not decide.** Business logic belongs in the domain.

**8. Tests before code, at every layer.** The acceptance test before any implementation. Domain unit tests before aggregates. Interactor tests before interactors. Integration tests before adapters.

**9. Implementation prompts reference test files.** The AI makes specific, pre-existing tests pass. Without this constraint, the AI optimises for tests that are easy to pass.

**10. The dependency rule is non-negotiable.** If the domain imports anything from an outer layer, the architecture is broken. The fix is always to extract a port interface.

**11. When a use case changes, everything changes.** The use case is the single source of truth. A change triggers regeneration of the affected tests, designs, and technical designs. Manual patching of downstream artifacts is how a codebase drifts from its specification — and how the Threadline breaks.

---

*Threadline was developed by synthesising six source methodologies: use-case-driven design, Clean Architecture with Domain-Driven Design, test-driven development from specifications, Excalidraw flow diagramming, Figma design structuring, and AI-assisted agentic code generation. The worked example throughout is FieldBase CMS — an Airtable-style content management system with custom page and form building. The ten skills that automate the pipeline are available as SKILL.md files for use with any AI agent that supports skill-based invocation.*
