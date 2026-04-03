---
name: use-case-to-technical-design
description: >
  Use this skill when the user wants to derive a Clean Architecture and Domain-Driven Design technical design from a set of reviewed use cases. Triggers include 'design the architecture from these use cases', 'apply clean architecture to this spec', 'derive aggregates from these use cases', 'produce a technical design', 'map this to DDD', 'what are the bounded contexts', 'write the interactors', 'derive the domain model'. Also use automatically in the agentic pipeline after use-case-reviewer confirms all use cases are ready. Produces bounded context map, aggregate designs with invariants, domain events, typed errors, use case interactors, port interfaces, adapter sketches, directory structure, and contract test skeletons. Code examples in TypeScript and GoLang. Do NOT use before use-case-reviewer has confirmed all use cases are ready.
---

# Use Case to Technical Design

Derives a complete Clean Architecture + DDD technical design from a set of fully dressed, reviewed use cases.

## Overview

Clean Architecture defines four concentric layers (Entities, Use Cases, Interface Adapters, Frameworks & Drivers) but says nothing about what goes inside the domain. Domain-Driven Design fills that gap. Together they form a complete design system that maps directly from use case elements.

The derivation is deterministic: every part of a use case corresponds to a specific architectural concept. There is no invention required — only translation.

## Core mapping

Before producing any output, internalise this mapping table. Every design decision flows from it.

| Use case element | Architectural concept |
|---|---|
| Primary actor | Driving port (primary adapter calls the use case) |
| Goal | Public method on the use case interactor |
| Preconditions | Invariant guards on aggregates |
| Main success scenario steps | Method body of the interactor |
| Extensions — validation failures | Typed domain errors returned from aggregate methods |
| Extensions — alternate success paths | Domain events raised by the aggregate |
| Postconditions | Return type contract + integration test assertions |
| Stakeholders & interests | Non-functional constraints on the adapter layer |

---

## Step 1: Identify bounded contexts

Group use cases into bounded contexts by asking: what changes together? What speaks the same ubiquitous language?

### Rules for grouping

- Use cases sharing the same primary aggregate belong in the same context
- Use cases with different primary actors almost always belong in different contexts
- If two use cases share a noun but mean different things by it (e.g. "record" in a submission context vs. an audit context), that noun lives in both contexts under different names — do not unify them
- A shared kernel is a small set of types shared across contexts without duplication (e.g. UserId, WorkspaceId)

### Output

Produce a bounded context map:

```
┌──────────────────────────────────────┐
│  [Context Name]                      │
│  Use cases: UC-XX, UC-XX             │
│  Primary aggregate: [Name]           │
│  Key concepts: [noun list]           │
└──────────────────────────────────────┘
```

Draw context relationships:
- **Shared kernel** — small set of types used by multiple contexts unchanged
- **Customer/supplier** — one context produces events the other consumes
- **Anti-corruption layer** — one context translates the other's model at the boundary

---

## Step 2: Derive aggregates from use case nouns and preconditions

### Finding aggregates

Scan the use case for nouns that:
1. Appear in preconditions (they must exist before the scenario starts)
2. Are created in postconditions (they come into existence as a result)
3. Are the subject of extension conditions (they can be in different states)

Each such noun is a candidate aggregate root. Smaller nouns that only exist in the context of a larger one are entities or value objects inside that aggregate.

### Aggregate design rules (DDD)

- An aggregate is a consistency boundary. Everything inside it changes together in one transaction.
- Only the aggregate root is referenced from outside. Other entities inside are accessed through the root.
- Aggregates reference each other by ID only — never by object reference.
- All invariant enforcement happens inside the aggregate — not in the use case interactor, not in the controller.

### Mapping preconditions to invariants

Each precondition that references an aggregate's state becomes an invariant check at the start of the relevant aggregate method:

```
Precondition: "A form linked to the target table has been published"
→ Form.submit() checks: if (!this.status.isPublished()) throw FormNotPublishedError
```

### TypeScript pattern — aggregate root

```typescript
// domain/[context]/[AggregateName].ts

export class AggregateName {
  private readonly _events: DomainEvent[] = []

  private constructor(
    public readonly id: AggregateId,
    // ... other fields as value objects
  ) {}

  // Factory — for creating new instances
  static create(params: CreateParams): AggregateName {
    const instance = new AggregateName(AggregateId.generate(), ...)
    instance.raise(new AggregateCreated(instance.id))
    return instance
  }

  // Factory — for reconstituting from persistence
  static reconstitute(data: AggregateData): AggregateName {
    return new AggregateName(new AggregateId(data.id), ...)
  }

  // Domain behaviour — enforces invariants, raises events
  doSomething(input: ValueObject): Result<OutputType, DomainError> {
    // Precondition check (from use case preconditions)
    if (!this.canDoSomething()) {
      return err(new InvariantViolationError())
    }
    // ... behaviour
    this.raise(new SomethingHappened(this.id, ...))
    return ok(output)
  }

  domainEvents(): DomainEvent[] { return [...this._events] }
  clearEvents(): void { this._events.length = 0 }
  private raise(event: DomainEvent): void { this._events.push(event) }
}
```

### GoLang pattern — aggregate root

```go
// domain/[context]/[aggregate_name].go

type AggregateName struct {
    id     AggregateID
    // ... other fields
    events []DomainEvent
}

// Create — new instance
func NewAggregateName(params CreateParams) (*AggregateName, error) {
    a := &AggregateName{id: NewAggregateID()}
    a.raise(&AggregateCreated{ID: a.id})
    return a, nil
}

// Reconstitute — from persistence
func ReconstitueAggregateName(data AggregateData) *AggregateName {
    return &AggregateName{id: AggregateID(data.ID), ...}
}

// Domain behaviour
func (a *AggregateName) DoSomething(input ValueObject) (*OutputType, error) {
    if !a.canDoSomething() {
        return nil, &InvariantViolationError{ID: a.id}
    }
    // ... behaviour
    a.raise(&SomethingHappened{ID: a.id})
    return output, nil
}

func (a *AggregateName) DomainEvents() []DomainEvent { return a.events }
func (a *AggregateName) ClearEvents()                { a.events = nil }
func (a *AggregateName) raise(e DomainEvent)         { a.events = append(a.events, e) }
```

---

## Step 3: Map extensions to typed errors and domain events

### Rule: error vs. event

| Extension type | Mapping |
|---|---|
| Validation failure | Typed error returned from aggregate method |
| Precondition not met | Typed error returned from aggregate method |
| Alternate success path | Domain event raised by aggregate |
| Notification (owner notified, email sent) | Domain event → application-layer event handler |
| State change on another aggregate | Domain event → cross-context handler |

### TypeScript — typed error union per use case

```typescript
// domain/[context]/errors.ts

// One type per extension. Never use generic Error.
export type UseCaseError
  = PreconditionNotMetError       // from precondition
  | RequiredFieldMissingError     // from ext Xa
  | TypeValidationError           // from ext Xb
  | PermissionDeniedError         // from ext Xc

export class RequiredFieldMissingError {
  readonly _tag = 'RequiredFieldMissingError'
  constructor(public readonly fieldId: FieldId) {}
}
```

### GoLang — sentinel error types

```go
// domain/[context]/errors.go

type RequiredFieldMissingError struct { FieldID FieldID }
func (e *RequiredFieldMissingError) Error() string {
    return fmt.Sprintf("required field missing: %s", e.FieldID)
}

type PermissionDeniedError struct { ActorID ActorID; Resource string }
func (e *PermissionDeniedError) Error() string {
    return fmt.Sprintf("permission denied: %s on %s", e.ActorID, e.Resource)
}
```

### Domain event per alternate success extension

```typescript
// domain/[context]/events.ts

// One event per significant outcome (postconditions + alternate success extensions)
export class RecordCreated implements DomainEvent {
  readonly occurredAt = new Date()
  constructor(
    public readonly recordId: RecordId,
    public readonly tableId: TableId,
    public readonly status: RecordStatus,
  ) {}
}

// Application-layer handler — wired at startup, not imported by domain
export class NotifyOwnerOnPendingRecord {
  constructor(private readonly notifier: OwnerNotifier) {}

  async handle(event: RecordCreated): Promise<void> {
    if (!event.status.isPending()) return
    await this.notifier.notifyPendingRecord(event.tableId, event.recordId)
  }
}
```

---

## Step 4: Write the use case interactor

The interactor is a direct translation of the main success scenario steps. Each step becomes one line or block. Comments reference the step number.

### Rules

- One interactor class/struct per use case
- One public method: `execute(command)` → `Result | error`
- The interactor **orchestrates** — it calls domain methods and repositories, but contains no business logic itself
- Imports only: domain types, port interfaces. Never imports adapters, HTTP, or database packages
- Every extension error that surfaces from the domain is propagated — not swallowed

### TypeScript pattern

```typescript
// application/[context]/[UseCaseName]UseCase.ts

export interface [UseCaseName]Command {
  // Fields derived from: use case inputs (what the actor provides)
}

export interface [UseCaseName]Result {
  // Fields derived from: use case postconditions (what the actor receives)
}

export class [UseCaseName]UseCase {
  constructor(
    private readonly repo: AggregateRepository,    // port
    private readonly events: DomainEventPublisher, // port
    // ... other ports
  ) {}

  async execute(
    cmd: [UseCaseName]Command,
  ): Promise<Result<[UseCaseName]Result, UseCaseError>> {

    // Step N: [paste step text as comment]
    const aggregate = await this.repo.findById(new AggregateId(cmd.id))
    if (!aggregate) return err(new NotFoundError(cmd.id))

    // Step N+1: [paste step text]
    const result = aggregate.doSomething(ValueObject.from(cmd.input))
    if (result.isErr()) return result // propagate ext Xa, Xb errors

    // Step N+2: persist
    await this.repo.save(aggregate)

    // Step N+3: publish events (ext handlers fire here)
    await this.events.publishAll(aggregate.domainEvents())

    // Postcondition: [paste postcondition as comment]
    return ok({ id: aggregate.id.value, ... })
  }
}
```

### GoLang pattern

```go
// application/[context]/[use_case_name].go

type [UseCaseName]Command struct {
    // Fields from use case inputs
}

type [UseCaseName]Result struct {
    // Fields from use case postconditions
}

type [UseCaseName]UseCase struct {
    repo   AggregateRepository    // port
    events DomainEventPublisher   // port
}

func (uc *[UseCaseName]UseCase) Execute(cmd [UseCaseName]Command) (*[UseCaseName]Result, error) {
    // Step N: [paste step text]
    aggregate, err := uc.repo.FindByID(AggregateID(cmd.ID))
    if err != nil { return nil, err }

    // Step N+1: [paste step text]
    // Extensions Xa, Xb surface as typed errors
    if err := aggregate.DoSomething(NewValueObject(cmd.Input)); err != nil {
        return nil, err
    }

    if err := uc.repo.Save(aggregate); err != nil {
        return nil, err
    }

    if err := uc.events.PublishAll(aggregate.DomainEvents()); err != nil {
        return nil, err
    }

    return &[UseCaseName]Result{ID: string(aggregate.ID())}, nil
}
```

---

## Step 5: Define ports

Ports are interfaces defined in the application layer and implemented in the adapter layer. Every external dependency (database, email, message queue, clock, ID generator) is a port.

### Rules

- Ports are defined **next to the use case that needs them** in the application layer
- Port method signatures use **domain types**, not primitive strings or ints
- Port names describe the capability, not the implementation: `FormRepository` not `PostgresFormRepository`
- The application layer never imports an adapter

### Output format

For each use case, produce a `ports.ts` or `ports.go` file listing every interface the interactor depends on, with method signatures using domain types.

---

## Step 6: Sketch adapters

For each port, identify the concrete adapter that will implement it and sketch its structure.

### Driving adapters (primary — the actor enters here)

- HTTP controller: translates HTTP request → command, result → HTTP response
- Each typed domain error maps to a specific HTTP status code
- Extension errors that change the response body or status are handled here

### Driven adapters (secondary — called by the use case)

- Repository: implements the repository port using the chosen database
- Notifier: implements the notifier port using the chosen email/push service
- Event publisher: implements the event publisher port using the chosen bus

### Rule

Adapters know about the outside world (HTTP, SQL, SMTP). The domain and application layers never know about them. The dependency arrow points inward.

---

## Step 7: Produce the directory structure

Derive the directory structure from the bounded contexts. Every context gets its own folder at every layer.

```
src/                          (TypeScript) or internal/ (Go)
├── domain/
│   └── [context]/
│       ├── [AggregateName].ts
│       ├── [ValueObject].ts
│       ├── errors.ts
│       └── events/
│           └── [EventName].ts
├── application/
│   └── [context]/
│       ├── [UseCaseName]UseCase.ts
│       ├── [EventHandlerName].ts   ← one per alternate-success extension
│       └── ports.ts
├── adapters/
│   ├── http/
│   │   └── [UseCaseName]Controller.ts
│   ├── [database]/
│   │   └── [AggregateName]Repository.ts
│   └── [service]/
│       └── [PortName]Impl.ts
└── infrastructure/
    ├── DomainEventBus.ts
    ├── database.ts
    └── server.ts
```

---

## Step 8: Produce contract test skeletons

Postconditions become integration test assertions. Extensions become named test cases.

### Rules

- Test names follow: `UC-XX_step_N_[description]` for happy path, `UC-XX_ext_Na_[description]` for extensions
- Each test sets up state from the preconditions
- Each test asserts the postconditions — not internal state
- Extension tests assert the observable outcome of the extension, not the error type alone

### Output

For each use case, produce a skeleton test file with:
1. One happy path test asserting all postconditions
2. One test per extension asserting its observable outcome
3. Setup helpers derived from preconditions

---

## Output checklist

Produce all of the following for each bounded context:

- [ ] Bounded context map with context relationships
- [ ] Aggregate designs with: invariant checks from preconditions, typed errors from extensions, domain events from alternate success paths
- [ ] Value object types for all significant domain nouns
- [ ] One interactor per use case with step comments
- [ ] `ports.ts` / `ports.go` per use case grouping
- [ ] Adapter sketches: one driving (HTTP), one driven (repository) per context
- [ ] Directory structure
- [ ] Contract test skeletons: one per use case, one test per extension

## Dependency rule verification

Before finalising, verify the dependency rule holds. Run this check mentally or via linting:

- `domain/` imports: nothing outside domain
- `application/` imports: domain only
- `adapters/` imports: application and domain
- `infrastructure/` imports: adapters, application, domain

If any layer imports from a layer outside it, the rule is violated. Flag the violation and propose the correct fix (usually: extract an interface / port).
