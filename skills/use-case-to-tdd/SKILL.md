---
name: use-case-to-tdd
description: >
  Use this skill when the user wants to derive TDD test suites from reviewed use cases. Triggers include 'write tests from these use cases', 'derive the test suite', 'generate acceptance tests', 'create domain unit tests', 'write interactor tests', 'produce integration tests', 'TDD from this spec'. Also use automatically in the agentic pipeline (Phase 3A) after use-case-reviewer confirms all use cases are READY. Produces four levels of tests — all RED (failing) — before any implementation exists. Do NOT use before use-case-reviewer has confirmed all use cases are READY.
---

# Use Case to TDD

Derives a complete four-level test suite from reviewed use cases. All tests are written RED — no implementation exists yet. The tests define the target; the implementation is what satisfies the target.

## Core Principle

Every element of a fully dressed use case maps to a specific test at a specific level. Extensions are not optional — every extension is a test case. An extension without a test is an untested code path.

## The Double Loop

The **outer loop** is a BDD acceptance test covering the full user journey. It stays RED until every layer is implemented.

The **inner loops** are unit and integration tests, one per architectural layer. Each inner test is also written before its implementation. Together they make the outer loop go green.

```
┌────────────────────────────────────────────────┐
│  OUTER LOOP: Acceptance test (BDD)             │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │  Inner 1: Domain unit tests              │  │
│  │  Source: extensions + invariants          │  │
│  └──────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────┐  │
│  │  Inner 2: Interactor tests               │  │
│  │  Source: MSS steps (call sequence)        │  │
│  └──────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────┐  │
│  │  Inner 3: Integration / contract tests   │  │
│  │  Source: postconditions + ports           │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
```

## Mapping: Use Case Element → Test

| Use case element | Test level | Test construct |
|---|---|---|
| Preconditions | All levels | `Given` clause / `beforeEach` setup |
| Main success scenario | Interactor test | Mock call sequence verification |
| Extensions — validation | Domain unit test | Aggregate returns typed error |
| Extensions — alternate success | Integration test | Domain event raised, handler executed |
| Postconditions | Acceptance + integration | `Then` clause / final assertions |
| Full scenario | Acceptance test | Complete BDD scenario |

## Step 1: Write Acceptance Tests (Outer Loop)

One scenario per happy path. One scenario per extension.

### Gherkin derivation

```gherkin
Feature: [Use case title]

  Background:
    # From: shared preconditions
    Given [precondition 1]
    And [precondition 2]

  Scenario: [UC-ID] happy path — [goal]
    Given [scenario-specific precondition]
    When [actor action from triggering step]
    Then [postcondition 1]
    And [postcondition 2]

  Scenario: [UC-ID] ext Xa — [extension condition]
    Given [setup for extension condition]
    When [same triggering action]
    Then [extension outcome]
    And [no unintended side effects]
```

### Implementation pattern (TypeScript)

```typescript
// tests/acceptance/UC-XX-[Slug].acceptance.test.ts

describe('UC-XX: [Use case title]', () => {
  let app, db, spies

  beforeEach(async () => {
    db = await testDatabase()
    // Wire full app with test doubles for external services
    app = createApp({ db, ...spies })
  })

  afterEach(() => db.reset())

  // Happy path — from postconditions
  it('UC-XX: [postcondition summary]', async () => {
    // Given — from preconditions
    // When — from triggering step
    // Then — from postconditions (executable assertions)
  })

  // Extension Xa — one test per extension
  it('UC-XX ext Xa: [extension outcome]', async () => {
    // Given — setup for extension condition
    // When — same trigger
    // Then — extension outcome + no side effects
  })
})
```

### Implementation pattern (GoLang)

```go
// tests/acceptance/[slug]_test.go

func TestUCXX_HappyPath(t *testing.T) {
    db  := testDB(t)
    srv := newTestServer(t, db)
    // Given — preconditions via builders
    // When — HTTP call
    // Then — assertions on response + DB state
}

func TestUCXX_ExtXa_[Description](t *testing.T) {
    // Same structure, extension-specific setup
}
```

## Step 2: Write Domain Unit Tests (Inner Loop 1)

Test aggregates and value objects. Pure — no DB, no HTTP, no I/O. Each extension maps to at least one domain test.

### Rules

- Preconditions → invariant violation tests (aggregate rejects invalid state)
- Validation extensions → typed error returned by aggregate method
- Alternate success extensions → domain event raised
- Happy path → aggregate returns expected result + events

### Naming convention

```
test_uc[ID]_[step|ext|precondition]_[N]_[description]
```

### Pattern

```typescript
// tests/unit/domain/[context]/[Aggregate].test.ts

describe('[Aggregate].[method]() — UC-XX', () => {
  // Precondition violation
  it('UC-XX precondition: returns [Error] when [precondition not met]', () => {
    const agg = Aggregate.reconstitute(invalidStateData())
    const result = agg.method(validInput())
    expect(result.isErr()).toBe(true)
    expect(result.error).toBeInstanceOf(SpecificError)
  })

  // Extension Xa: validation
  it('UC-XX ext Xa: returns [ValidationError] when [condition]', () => {
    const agg = Aggregate.reconstitute(validStateData())
    const result = agg.method(invalidInput())
    expect(result.isErr()).toBe(true)
  })

  // Happy path
  it('UC-XX happy path: returns [Result] with [domain event]', () => {
    const agg = Aggregate.reconstitute(validStateData())
    const result = agg.method(validInput())
    expect(result.isOk()).toBe(true)
    expect(result.value.domainEvents()).toContainEqual(
      expect.any(ExpectedEvent)
    )
  })
})
```

## Step 3: Write Interactor Tests (Inner Loop 2)

Test use case orchestration. All ports are mocked. Verify the call sequence matches the MSS steps.

### Rules

- One class per use case, one `execute` method
- Mock every port dependency
- Verify call sequence: step comments reference MSS step numbers
- Domain errors propagate unchanged — verify no side effects on error

### Pattern

```typescript
// tests/unit/application/[context]/[UseCase]UseCase.test.ts

describe('[UseCase]UseCase — UC-XX', () => {
  let mocks, useCase

  beforeEach(() => {
    mocks = { repo: new MockRepo(), events: new MockEventPublisher() }
    useCase = new UseCaseClass(mocks.repo, mocks.events)
  })

  it('UC-XX happy path: [step sequence description]', async () => {
    mocks.repo.willReturn(validFixture())
    const result = await useCase.execute(validCommand())
    expect(result.isOk()).toBe(true)
    expect(mocks.repo.findByIdCalled).toBe(true)   // step N
    expect(mocks.repo.saveCalled).toBe(true)        // step M
    expect(mocks.events.publishAllCalled).toBe(true) // step K
  })

  it('UC-XX ext Xa: [Error] propagated, no side effects', async () => {
    mocks.repo.willReturn(fixtureTriggering_Xa())
    const result = await useCase.execute(commandTriggering_Xa())
    expect(result.isErr()).toBe(true)
    expect(mocks.repo.saveCalled).toBe(false)
    expect(mocks.events.publishAllCalled).toBe(false)
  })
})
```

## Step 4: Write Integration / Contract Tests (Inner Loop 3)

Test adapter implementations against port contracts using real infrastructure (Testcontainers).

### Rules

- One test file per adapter
- Tests verify the adapter implements the port interface correctly
- Use real database (Testcontainers), real event bus
- Not full-stack — that's the acceptance test's job

### Pattern

```typescript
// tests/integration/adapters/[adapter]/[Repository].test.ts

describe('[Adapter][Repository] — contract test ([Port] port)', () => {
  let repo, db

  beforeAll(async () => {
    db = await startPostgresContainer()
    await runMigrations(db)
    repo = new AdapterRepository(db)
  })

  afterEach(() => db.query('DELETE FROM [table]'))
  afterAll(() => db.end())

  it('findById returns null when not found', async () => {
    expect(await repo.findById(Id.create('missing'))).toBeNull()
  })

  it('findById returns reconstituted aggregate', async () => {
    await seedData(db, { id: 'x', status: 'published' })
    const result = await repo.findById(Id.create('x'))
    expect(result).not.toBeNull()
    expect(result!.status.isPublished()).toBe(true)
  })
})
```

## Step 5: Produce Test Helper Stubs

For each use case, produce stubs for:

| Helper type | Naming | Purpose |
|---|---|---|
| Builder | `[Entity]Builder` | Fluent setup for acceptance/integration (uses real DB) |
| Fixture | `[state][Entity]Data()` | In-memory aggregate data for domain unit tests |
| Spy | `[Port]Spy` | Records calls for acceptance tests |
| Mock | `Mock[Port]` | Returns configured values for interactor tests |
| Seed | `seed[Entity](db, data)` | Direct DB insert for integration tests |

## Output

For each use case, produce:

```
tests/
├── acceptance/
│   └── UC-XX-[Slug].acceptance.test.[ts|go]
├── unit/
│   ├── domain/[context]/[Aggregate].test.[ts|go]
│   └── application/[context]/[UseCase]UseCase.test.[ts|go]
├── integration/
│   └── adapters/[adapter]/[Repository].test.[ts|go]
└── helpers/
    ├── builders/[Entity]Builder.[ts|go]
    ├── fixtures/[entity]Fixtures.[ts|go]
    ├── spies/[Port]Spy.[ts|go]
    └── mocks/Mock[Port].[ts|go]
```

### Verification checklist

- [ ] One acceptance scenario per happy path
- [ ] One acceptance scenario per extension
- [ ] One domain unit test per extension (validation → typed error)
- [ ] One domain unit test per precondition violation
- [ ] Happy path domain test returns result + domain event
- [ ] Interactor test verifies MSS call sequence with step comments
- [ ] Interactor test verifies error propagation with no side effects
- [ ] Integration test per adapter/port contract
- [ ] All test names reference UC-ID and step/extension number
- [ ] All tests are RED — no implementation exists yet
- [ ] Helper stubs are scaffolded (builders, fixtures, spies, mocks)

## Common Mistakes

- **Writing tests that pass immediately** — if a test passes without implementation, it's testing the wrong thing or the setup is wrong. All tests must be RED.
- **Skipping extension tests** — every extension is a test. No exceptions.
- **Testing implementation details in domain tests** — test behaviour (method returns error, raises event), not internal state.
- **Using real infrastructure in interactor tests** — interactor tests mock everything. Real infra belongs in integration and acceptance tests only.
- **Naming tests without UC references** — test names must include UC-ID and step/extension number for traceability.
