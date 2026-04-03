# TDD, Clean Architecture, and the Agentic Pipeline
## A Complete Guide to Test-Driven Development from Use Cases

---

## Introduction

This guide explains how Test-Driven Development integrates into the use-case-driven development process. It covers the double loop model, the four test levels mapped to the four architectural layers, how each use case element maps to a specific test, and how the pipeline orchestrator skill coordinates all nine sub-skills into a single coherent workflow.

The central argument: use cases are not just a specification tool. They are the source of truth for every test, every design decision, and every line of code. TDD makes that traceability explicit by forcing you to write the test before the code — and the test is always derived from the use case.

---

## Part 1: Why TDD Changes the Pipeline Order

Without TDD the flow is: spec → design → implement → test. The test is an afterthought that verifies something you already built. With TDD the flow becomes:

```
spec → test (red) → implement (green) → refactor → repeat
```

This changes the pipeline in a specific way: tests are written immediately after the use cases are reviewed, before any implementation or even any detailed technical design. The tests define the target; the implementation is what satisfies the target.

In the use-case-driven pipeline, this produces the following phase order:

```
Phase 1: Write use cases
Phase 2: Review use cases (Gate 1)
Phase 3A: Write TDD test suites (red — all failing)
Phase 3B: Write technical design (aggregates, interactors, ports, adapters)
Phase 3C: Write design artefacts (Excalidraw, Figma)
Phase 4: Human review (Gate 2)
Phase 5: Generate AI implementation prompts (make the tests go green)
```

The implementation prompt in Phase 5 references the specific test files from Phase 3A. The AI is instructed to make those tests pass — not to invent its own interpretation of the use case.

---

## Part 2: The Double Loop

The double loop is the key structural concept of TDD in a layered architecture.

The **outer loop** is a BDD acceptance test. It covers the complete user journey: an HTTP request in, a database state verified out. It is written first and stays red until every layer is implemented. It defines "done" for the entire use case.

The **inner loop** is a series of unit and integration tests, one per layer. Each inner loop test is also written before its implementation. The inner loops collectively make the outer loop go green.

```
┌──────────────────────────────────────────────────────┐
│  OUTER LOOP (BDD acceptance test)                    │
│                                                      │
│  Given [preconditions]                               │
│  When  [actor action from step 1 or triggering step] │
│  Then  [postconditions]                              │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  INNER LOOP 1: Domain unit tests             │   │
│  │  Tests aggregates, value objects             │   │
│  │  Derived from: extensions + invariants       │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  INNER LOOP 2: Interactor tests              │   │
│  │  Tests use case orchestration                │   │
│  │  Derived from: main success scenario steps   │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  INNER LOOP 3: Integration/contract tests    │   │
│  │  Tests adapters against port contracts       │   │
│  │  Derived from: postconditions                │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## Part 3: The Two TDD Schools, Reconciled

**Outside-In (London school):** Start at the acceptance test. Work inward. Mock everything on the inner boundary until you implement it. You discover the API of each layer by asking "what does the layer above need from me?"

**Inside-Out (Detroit/Chicago school):** Start at the domain core. Build outward with real objects. You discover the domain model by working with it directly.

For this process, use both in sequence:

1. Write the acceptance test first (outside-in — defines the target)
2. Drop to the domain and use inside-out TDD to build aggregates
3. Rise to the interactor using outside-in with mocked ports
4. Rise to the adapter layer using integration tests
5. The acceptance test goes green — the use case is complete

This sequence follows the dependency rule: you always implement from the inside out, but you define the target from the outside in.

---

## Part 4: Mapping Use Case Elements to Tests

Every element of a fully dressed use case corresponds to a specific test or test clause.

| Use case element | Test level | Test construct |
|---|---|---|
| Preconditions | All levels | `Given` clause / `beforeEach` setup |
| Main success scenario | Interactor test | Mock call sequence verification |
| Extensions — validation | Domain unit test | Aggregate returns typed error |
| Extensions — alternate success | Integration test | Domain event raised, handler executed |
| Postconditions | Acceptance + integration | `Then` clause / final assertions |
| Full scenario | Acceptance test | Complete BDD scenario |

---

## Part 5: Level 1 — Acceptance Tests

Acceptance tests are the outer loop. Write them first. They define what "done" means for each use case.

### Gherkin derivation

Map each use case clause directly to Gherkin:

```gherkin
Feature: [Use case title]

  Background:
    # From: preconditions that are shared across scenarios
    Given [precondition 1]
    And [precondition 2]

  Scenario: [Use case ID] happy path — [goal]
    # From: preconditions specific to this scenario
    Given [scenario-specific precondition]
    # From: the triggering step (usually step 1 or step 3)
    When [actor action]
    # From: postconditions
    Then [observable outcome 1]
    And [observable outcome 2]

  Scenario: [Use case ID] ext Xa — [extension condition]
    Given [setup for extension condition]
    When [same triggering action]
    Then [extension observable outcome]
    And [no unintended side effects — e.g. no record created]
```

One scenario per main success scenario. One scenario per extension.

### TypeScript — acceptance test implementation

```typescript
// tests/acceptance/UC-02-SubmitForm.acceptance.test.ts

describe('UC-02: Submit data via a custom form', () => {
  let app: Express
  let db: TestDatabase
  let notifierSpy: NotifierSpy

  beforeEach(async () => {
    db = await testDatabase()
    notifierSpy = new NotifierSpy()
    // Wire the full application with test doubles for external services
    app = createApp({ db, ownerNotifier: notifierSpy })
  })

  afterEach(() => db.reset())

  // Happy path
  it('UC-02: creates an active record on valid submission', async () => {
    // Given — preconditions
    const table = await TableBuilder.create(db, { name: 'Events' })
    const form  = await FormBuilder.publishedForm(db, {
      tableId: table.id,
      fields: [requiredTextField('name'), requiredDateField('date')],
    })

    // When — triggering step
    const res = await request(app)
      .post(`/forms/${form.id}/submissions`)
      .send({ name: 'React Summit', date: '2025-06-14' })

    // Then — postconditions
    expect(res.status).toBe(201)
    const record = await db.records.findById(res.body.recordId)
    expect(record).not.toBeNull()
    expect(record!.status).toBe('active')
  })

  // Extension 4a
  it('UC-02 ext 4a: required field empty returns 422, no record created', async () => {
    const form = await FormBuilder.publishedWithRequiredField(db, 'name')

    const res = await request(app)
      .post(`/forms/${form.id}/submissions`)
      .send({ date: '2025-06-14' }) // name missing

    expect(res.status).toBe(422)
    expect(res.body.errors).toContainEqual(
      expect.objectContaining({ field: 'name', code: 'required' })
    )
    const records = await db.records.findByTableId(form.tableId)
    expect(records).toHaveLength(0)
  })

  // Extension 6a
  it('UC-02 ext 6a: approval mode writes pending and notifies owner', async () => {
    const form = await FormBuilder.publishedWithApproval(db)

    const res = await request(app)
      .post(`/forms/${form.id}/submissions`)
      .send({ name: 'React Summit' })

    expect(res.status).toBe(202)
    const record = await db.records.findById(res.body.recordId)
    expect(record!.status).toBe('pending')
    expect(notifierSpy.calls).toHaveLength(1)
  })
})
```

### GoLang — acceptance test

```go
// tests/acceptance/submit_form_test.go

func TestUC02_HappyPath(t *testing.T) {
    db  := testDB(t)
    spy := &NotifierSpy{}
    srv := newTestServer(t, db, spy)

    table := tableBuilder(t, db).Named("Events").Create()
    form  := formBuilder(t, db).ForTable(table.ID).Published().
               WithField(requiredTextField("name")).
               WithField(requiredDateField("date")).Create()

    res := srv.POST(fmt.Sprintf("/forms/%s/submissions", form.ID),
        body("name", "React Summit", "date", "2025-06-14"))

    assert.Equal(t, 201, res.Code)
    record := findRecord(t, db, jsonField(res, "recordId"))
    assert.Equal(t, "active", string(record.Status()))
}

func TestUC02_Ext4a_RequiredFieldMissing(t *testing.T) {
    db   := testDB(t)
    form := publishedFormWithRequiredField(t, db, "name")
    srv  := newTestServer(t, db, &NotifierSpy{})

    res := srv.POST(fmt.Sprintf("/forms/%s/submissions", form.ID),
        body("date", "2025-06-14"))

    assert.Equal(t, 422, res.Code)
    assert.Empty(t, findRecordsByTable(t, db, form.TableID))
}

func TestUC02_Ext6a_ApprovalMode(t *testing.T) {
    db   := testDB(t)
    spy  := &NotifierSpy{}
    form := publishedFormWithApproval(t, db)
    srv  := newTestServer(t, db, spy)

    res := srv.POST(fmt.Sprintf("/forms/%s/submissions", form.ID),
        body("name", "React Summit"))

    assert.Equal(t, 202, res.Code)
    assert.Equal(t, "pending", string(findRecord(t, db, jsonField(res, "recordId")).Status()))
    assert.Len(t, spy.Calls, 1)
}
```

---

## Part 6: Level 2 — Domain Unit Tests

Domain unit tests cover aggregates and value objects. They are fast, pure, and require no infrastructure. Each extension in the use case becomes at least one domain unit test.

Write them before implementing the aggregate. The test defines the interface — what the aggregate method accepts, what it returns when the condition is met.

```typescript
// tests/unit/domain/submission/Form.test.ts

describe('Form.submit() — UC-02', () => {

  // Precondition: form must be published
  describe('when form is not published', () => {
    it('UC-02 precondition: returns FormNotPublishedError', () => {
      const form = Form.reconstitute(draftFormData())
      const result = form.submit(validFieldValues())
      expect(result.isErr()).toBe(true)
      expect(result.error).toBeInstanceOf(FormNotPublishedError)
    })
  })

  // Extension 4a: required field missing
  describe('ext 4a: required field empty', () => {
    it('returns ValidationError with code "required"', () => {
      const form = Form.reconstitute(publishedFormWith([
        { id: 'name', type: 'text', required: true }
      ]))
      const result = form.submit(FieldValues.from({ date: '2025-06-14' }))
      expect(result.isErr()).toBe(true)
      expect((result.error as ValidationError).errors).toContainEqual(
        expect.objectContaining({ fieldId: 'name', code: 'required' })
      )
    })
  })

  // Extension 4b: type mismatch
  describe('ext 4b: type mismatch', () => {
    it('returns ValidationError with code "type_mismatch" for invalid date', () => {
      const form = Form.reconstitute(publishedFormWith([
        { id: 'date', type: 'date', required: true }
      ]))
      const result = form.submit(FieldValues.from({ date: 'not-a-date' }))
      expect((result.error as ValidationError).errors).toContainEqual(
        expect.objectContaining({ code: 'type_mismatch' })
      )
    })
  })

  // Happy path
  describe('happy path: valid values on published form', () => {
    it('returns a Record', () => {
      const form = Form.reconstitute(publishedFormData())
      const result = form.submit(validFieldValues())
      expect(result.isOk()).toBe(true)
    })

    it('raises RecordCreated domain event', () => {
      const form = Form.reconstitute(publishedFormData())
      const result = form.submit(validFieldValues())
      const events = result.value.domainEvents()
      expect(events.some(e => e instanceof RecordCreated)).toBe(true)
    })

    // Extension 6a
    it('ext 6a: record is Pending when approval required', () => {
      const form = Form.reconstitute(publishedFormWith([], { requiresApproval: true }))
      const result = form.submit(validFieldValues())
      expect(result.value.status.isPending()).toBe(true)
    })

    it('record is Active when no approval required', () => {
      const form = Form.reconstitute(publishedFormWith([], { requiresApproval: false }))
      const result = form.submit(validFieldValues())
      expect(result.value.status.isActive()).toBe(true)
    })
  })
})
```

### GoLang

```go
// tests/unit/domain/submission/form_test.go

func TestForm_Submit_PreconditionViolated(t *testing.T) {
    form := reconstituteForm(t, draftFormData())
    _, err := form.Submit(validFieldValues())
    var e *FormNotPublishedError
    assert.ErrorAs(t, err, &e)
}

func TestForm_Submit_Ext4a_RequiredFieldMissing(t *testing.T) {
    form := reconstituteForm(t, publishedFormWith(requiredTextField("name")))
    _, err := form.Submit(fieldValues(map[string]any{"date": "2025-06-14"}))
    var e *ValidationError
    require.ErrorAs(t, err, &e)
    assert.Equal(t, "required", fieldCode(e, "name"))
}

func TestForm_Submit_Ext4b_TypeMismatch(t *testing.T) {
    form := reconstituteForm(t, publishedFormWith(requiredDateField("date")))
    _, err := form.Submit(fieldValues(map[string]any{"date": "not-a-date"}))
    var e *ValidationError
    require.ErrorAs(t, err, &e)
    assert.Equal(t, "type_mismatch", fieldCode(e, "date"))
}

func TestForm_Submit_HappyPath(t *testing.T) {
    form := reconstituteForm(t, publishedFormData())
    record, err := form.Submit(validFieldValues())
    require.NoError(t, err)
    assert.True(t, containsEvent(record.DomainEvents(), &RecordCreated{}))
}

func TestForm_Submit_Ext6a_ApprovalMode(t *testing.T) {
    form := reconstituteForm(t, publishedFormWith(approval(true)))
    record, err := form.Submit(validFieldValues())
    require.NoError(t, err)
    assert.Equal(t, RecordStatusPending, record.Status())
}
```

---

## Part 7: Level 3 — Interactor Tests

Interactor tests verify that the use case orchestrates the domain and ports correctly. Ports are mocked. The test verifies the sequence of calls matches the main success scenario steps.

These tests use the London school approach: every dependency is a mock or spy. The test is not checking that the domain works (domain unit tests do that) — it is checking that the interactor calls the right things in the right order.

```typescript
// tests/unit/application/submission/SubmitFormUseCase.test.ts

describe('SubmitFormUseCase — UC-02', () => {
  let forms:   MockFormRepository
  let records: MockRecordRepository
  let events:  MockEventPublisher
  let useCase: SubmitFormUseCase

  beforeEach(() => {
    forms   = new MockFormRepository()
    records = new MockRecordRepository()
    events  = new MockEventPublisher()
    useCase = new SubmitFormUseCase(forms, records, events)
  })

  // Steps 3-6: happy path call sequence
  it('UC-02 happy path: loads form → submits → saves record → publishes events', async () => {
    forms.willReturn(publishedFormFixture())

    const result = await useCase.execute({
      formId: 'form-1',
      values: { name: 'React Summit', date: '2025-06-14' },
    })

    expect(result.isOk()).toBe(true)
    expect(forms.findByIdCalled).toBe(true)     // step 3
    expect(records.saveCalled).toBe(true)        // step 5
    expect(events.publishAllCalled).toBe(true)   // step 6
  })

  // Propagation: domain errors flow through the interactor unchanged
  it('UC-02 ext 4a: ValidationError from domain is propagated', async () => {
    forms.willReturn(publishedFormWithRequiredField('name'))

    const result = await useCase.execute({ formId: 'f', values: {} })

    expect(result.isErr()).toBe(true)
    expect(result.error).toBeInstanceOf(ValidationError)
    expect(records.saveCalled).toBe(false)
    expect(events.publishAllCalled).toBe(false)
  })

  // Precondition: form not found
  it('returns FormNotPublishedError when form does not exist', async () => {
    forms.willReturn(null)

    const result = await useCase.execute({ formId: 'missing', values: {} })

    expect(result.isErr()).toBe(true)
    expect(result.error).toBeInstanceOf(FormNotPublishedError)
  })
})
```

### GoLang

```go
// tests/unit/application/submission/submit_form_test.go

func TestSubmitFormUseCase_HappyPath_CallSequence(t *testing.T) {
    form    := publishedFormFixture()
    forms   := &MockFormRepo{Returns: form}
    records := &MockRecordRepo{}
    events  := &MockEventPublisher{}

    uc := &SubmitFormUseCase{Forms: forms, Records: records, Events: events}
    result, err := uc.Execute(SubmitFormCommand{
        FormID: string(form.ID()),
        Values: map[string]any{"name": "React Summit"},
    })

    require.NoError(t, err)
    assert.NotNil(t, result)
    assert.True(t, forms.FindByIDCalled)      // step 3
    assert.NotNil(t, records.Saved)            // step 5
    assert.NotEmpty(t, events.Published)       // step 6
}

func TestSubmitFormUseCase_Ext4a_PropagatesValidationError(t *testing.T) {
    forms   := &MockFormRepo{Returns: publishedFormWithRequiredField("name")}
    records := &MockRecordRepo{}
    events  := &MockEventPublisher{}

    uc := &SubmitFormUseCase{Forms: forms, Records: records, Events: events}
    _, err := uc.Execute(SubmitFormCommand{FormID: "id", Values: map[string]any{}})

    var valErr *ValidationError
    assert.ErrorAs(t, err, &valErr)
    assert.Nil(t, records.Saved)
    assert.Empty(t, events.Published)
}
```

---

## Part 8: Level 4 — Integration and Contract Tests

Integration tests verify that the adapter implementations fulfil the port contracts using real infrastructure (test database, in-memory event bus). They test one adapter at a time.

Contract tests answer: "Does this adapter actually implement the port interface correctly?" They are not full-stack tests — that is the acceptance test's job.

Use Testcontainers for GoLang or `@testcontainers/postgresql` for TypeScript to spin up a real database for each test run.

```typescript
// tests/integration/adapters/postgres/FormRepository.test.ts

describe('PostgresFormRepository — contract test (FormRepository port)', () => {
  let repo: PostgresFormRepository
  let db: Pool

  beforeAll(async () => {
    db = await startPostgresContainer()
    await runMigrations(db)
    repo = new PostgresFormRepository(db)
  })

  afterEach(() => db.query('DELETE FROM forms'))
  afterAll(() => db.end())

  it('findById returns null when form does not exist', async () => {
    const result = await repo.findById(FormId.create('nonexistent'))
    expect(result).toBeNull()
  })

  it('findById returns reconstituted Form matching what was seeded', async () => {
    await db.query(`
      INSERT INTO forms (id, table_id, status, policy, fields_json)
      VALUES ($1, $2, 'published', 'no_approval', '[]')
    `, ['form-1', 'table-1'])

    const form = await repo.findById(FormId.create('form-1'))

    expect(form).not.toBeNull()
    expect(form!.status.isPublished()).toBe(true)
    expect(form!.tableId.value).toBe('table-1')
  })
})
```

```go
// tests/integration/adapters/postgres/form_repository_test.go

func TestPostgresFormRepository_FindByID_NotFound(t *testing.T) {
    db   := testContainerDB(t)
    repo := postgres.NewFormRepository(db)

    result, err := repo.FindByID("nonexistent")
    require.NoError(t, err)
    assert.Nil(t, result)
}

func TestPostgresFormRepository_FindByID_ReconstitutesCorrectly(t *testing.T) {
    db := testContainerDB(t)
    seedForm(t, db, SeedFormData{
        ID: "form-1", TableID: "table-1", Status: "published",
    })
    repo := postgres.NewFormRepository(db)

    form, err := repo.FindByID("form-1")
    require.NoError(t, err)
    assert.True(t, form.Status().IsPublished())
    assert.Equal(t, "table-1", string(form.TableID()))
}
```

---

## Part 9: Test Helper Conventions

A consistent set of test helpers makes the suite maintainable and keeps test code focused on the scenario, not on setup plumbing.

### Builders (for acceptance and integration tests)

Fluent builders create domain objects in specific states using the real database:

```typescript
// tests/helpers/builders/FormBuilder.ts

export class FormBuilder {
  private fields: FormFieldData[] = []
  private requiresApproval = false
  private status: FormStatus = 'draft'

  static create(db: TestDatabase, opts?: Partial<FormBuilderOptions>): FormBuilder {
    return new FormBuilder(db, opts)
  }

  withField(field: FormFieldData): this {
    this.fields.push(field)
    return this
  }

  published(): this {
    this.status = 'published'
    return this
  }

  withApproval(): this {
    this.requiresApproval = true
    return this
  }

  async build(): Promise<FormRecord> {
    return this.db.forms.insert({
      fields: this.fields,
      status: this.status,
      requiresApproval: this.requiresApproval,
    })
  }
}
```

### Fixtures (for domain unit tests)

In-memory aggregate instances — no database, no I/O:

```typescript
// tests/helpers/fixtures/formFixtures.ts

export const publishedFormData = (overrides?: Partial<FormData>): FormData => ({
  id: 'form-fixture-1',
  tableId: 'table-fixture-1',
  status: 'published',
  policy: 'no_approval',
  fields: [
    { id: 'name', type: 'text', required: true },
    { id: 'date', type: 'date', required: true },
  ],
  ...overrides,
})

export const validFieldValues = (): FieldValues =>
  FieldValues.from({ name: 'React Summit', date: '2025-06-14' })
```

### Spies (for acceptance tests)

Port implementations that record calls without performing real I/O:

```typescript
// tests/helpers/spies/NotifierSpy.ts

export class NotifierSpy implements OwnerNotifier {
  calls: Array<{ tableId: string; recordId: string }> = []

  async notifyPendingRecord(tableId: TableId, recordId: RecordId): Promise<void> {
    this.calls.push({ tableId: tableId.value, recordId: recordId.value })
  }

  get wasNotified(): boolean {
    return this.calls.length > 0
  }
}
```

### Mocks (for interactor tests)

Port implementations that return configured values:

```typescript
// tests/helpers/mocks/MockFormRepository.ts

export class MockFormRepository implements FormRepository {
  private _returnValue: Form | null = null
  findByIdCalled = false

  willReturn(form: Form | null): void {
    this._returnValue = form
  }

  async findById(_id: FormId): Promise<Form | null> {
    this.findByIdCalled = true
    return this._returnValue
  }
}
```

### Naming convention summary

| Helper type | Naming pattern | Example |
|---|---|---|
| Builder | `[Entity]Builder.[method]()` | `FormBuilder.publishedWithApproval()` |
| Fixture | `[state][Entity]Data()` | `publishedFormData()`, `draftFormData()` |
| Spy | `[Port]Spy` | `NotifierSpy`, `EventPublisherSpy` |
| Mock | `Mock[Port]` | `MockFormRepository`, `MockEventPublisher` |
| Seed | `seed[Entity](db, data)` | `seedPublishedForm(db, { id: 'f-1' })` |

---

## Part 10: The Complete Pipeline

The nine skills form a directed acyclic graph with two human review gates and two parallel branches.

### The DAG

```
[Input: product description]
        │
        ▼
[1] use-case-writer
        │
        ▼
[2] use-case-reviewer ──────────────── GATE 1
        │
        │  (all use cases READY)
        │
        ├────────────────────┬──────────────────────┐
        │                    │                      │
        ▼                    ▼                      ▼
[3A] use-case-to-tdd   [3B] use-case-to-    [3C] screen-inventory
     │                      technical-design       │
     │ acceptance tests      │                     ├─[4A] excalidraw-generator
     │ domain unit tests     │ bounded contexts     │
     │ interactor tests      │ aggregates           └─[4B] component-inventory
     │ integration tests     │ interactors                  │
     │                       │ ports/adapters               ▼
     │                       │ directory struct         [4C] figma-spec
     │                       │
     └───────────────────────┴──────────────────────── GATE 2
                                                     (human review)
                                                           │
                                                           ▼
                                               [5] use-case-to-prompt
                                                    │
                                                    │ data model prompts
                                                    │ implementation prompts
                                                    │ test completion prompts
                                                    ▼
                                               [Output: handoff package]
```

### Gate 1: use-case-reviewer

All use cases must be marked READY before any downstream skill runs. The reviewer checks six dimensions: implementability, extension completeness, precondition completeness, postcondition observability, actor consistency, and goal level fit.

Do not proceed past Gate 1 if any use case is NOT READY. Fix and re-review.

### Gate 2: human review

At Gate 2, a human reviews four artefacts simultaneously:

- **Excalidraw diagrams** — verify every step and extension is present and correctly attributed to the right actor's swim lane
- **Technical design** — verify bounded contexts are correctly identified, aggregate invariants match preconditions, typed errors match extensions
- **TDD test skeletons** — verify every extension has a test, every postcondition has an assertion, all tests are red (no implementation yet)
- **Figma spec** — verify screen coverage, component variants, annotation conventions

If gaps are found in the use cases at Gate 2, return to Phase 1. Cascade the change through all downstream phases. Do not patch artefacts to paper over a use case gap.

### Parallel execution

Branches 3A, 3B, and 3C are independent. They can run concurrently if the execution environment supports parallel sub-agents. In a sequential environment, run 3A first (TDD tests drive design decisions), then 3B, then 3C.

### Phase 5: The TDD-aware generation prompt

The implementation prompt in Phase 5 is different from a standard generation prompt because it references the specific test files from Phase 3A. The AI is instructed to make those tests pass:

```
Implement [UseCase] in [language/framework] such that all tests in 
[tests/unit/domain/submission/Form.test.ts] and 
[tests/unit/application/submission/SubmitFormUseCase.test.ts] pass.

The technical design is in [technical-design/application/submission/SubmitFormUseCase.ts].
The acceptance test is in [tests/acceptance/UC-02-SubmitForm.acceptance.test.ts].

Do not modify any test files. Do not add any tests. Make the existing tests pass
by implementing the production code.

Implement in this order:
1. Value objects (FieldValues, FormId, RecordStatus)
2. Domain errors (ValidationError, FormNotPublishedError)
3. Domain events (RecordCreated)
4. Form aggregate
5. Record aggregate
6. SubmitFormUseCase interactor
7. Ports (FormRepository, RecordRepository, DomainEventPublisher)

Stop after step 7. Do not implement adapters — those will be a separate prompt.
```

This is the key distinction from a non-TDD prompt: the AI's output is constrained by pre-existing test files. The tests were derived from the use case; the code must satisfy the tests; the use case is therefore enforced transitively.

---

## Part 11: The Artifact Manifest

At the end of the full pipeline, the developer receives a complete, traceable package:

```
outputs/
├── use-cases.md                    ← single source of truth
│
├── tests/
│   ├── acceptance/
│   │   └── UC-XX-[slug].acceptance.test.ts   (× use case count)
│   ├── unit/
│   │   ├── domain/[context]/[Aggregate].test.ts
│   │   └── application/[context]/[UseCase]UseCase.test.ts
│   └── integration/
│       └── adapters/[adapter]/[Repository].test.ts
│
├── technical-design/
│   ├── bounded-contexts.md
│   ├── domain/[context]/[Aggregate].[ts|go]
│   ├── application/[context]/[UseCase]UseCase.[ts|go]
│   ├── application/[context]/ports.[ts|go]
│   ├── adapters/http/[UseCase]Controller.[ts|go]
│   ├── adapters/[db]/[Aggregate]Repository.[ts|go]
│   └── directory-structure.md
│
├── design/
│   ├── screen-inventory.md
│   ├── UC-XX-[slug].excalidraw   (× use case count)
│   ├── component-inventory.md
│   └── figma-spec.md
│
└── prompts/
    ├── UC-XX-data-model.md        (× use case count)
    ├── UC-XX-implementation.md    (× use case count)
    └── UC-XX-tests.md             (× use case count)
```

### The traceability chain

Every artefact in the package is traceable in both directions:

**Forward (spec to test to code):**
```
Use case extension 4a
  → Domain unit test: Form.test.ts line 28
    → Typed error: ValidationError (required)
      → Implementation: Form.submit() validation branch
        → HTTP response: 422 with errors array
          → Acceptance test assertion: res.body.errors
```

**Backward (bug to spec):**
```
Failing test: UC-02 ext 4a required field blocks submission
  → Test references: UC-02 extension 4a
    → Use case: Form submission, step 4, extension 4a
      → Spec says: "System highlights the field, blocks submission"
        → Design decision traceable to the use case
```

A bug whose test name references a use case ID is a bug whose cause is immediately locatable in the spec. That is the value of this process.

---

## Part 12: Invoking the Pipeline Orchestrator

To run the complete pipeline with a single prompt to the agent:

```
Read skills/pipeline-orchestrator/SKILL.md and run the complete pipeline.

Product description:
[description of what you want to build]

Language(s): [TypeScript | GoLang | both]

Framework context:
  HTTP: [Express | Fastify | net/http | Gin | Chi]
  ORM/DB: [Prisma | Drizzle | database/sql | GORM]
  Test framework: [Jest | Vitest | Go test | testify]
  Existing patterns: [any relevant conventions]

Available skill files:
  skills/pipeline-orchestrator/SKILL.md
  skills/use-case-writer/SKILL.md
  skills/use-case-reviewer/SKILL.md
  skills/use-case-to-tdd/SKILL.md
  skills/use-case-to-technical-design/SKILL.md
  skills/screen-inventory/SKILL.md
  skills/excalidraw-generator/SKILL.md
  skills/component-inventory/SKILL.md
  skills/figma-spec/SKILL.md
  skills/use-case-to-prompt/SKILL.md

Instructions:
- Pause at Gate 1 and present the review before proceeding
- Pause at Gate 2 and present all artefacts before proceeding
- All tests must be written RED — no implementation until Gate 2 is cleared
- Run branches 3A, 3B, and 3C in parallel if possible
- Produce the artifact manifest at the end
```

---

## Summary: The Ten Principles

1. **Use cases before tests.** You cannot write a meaningful test without a clear use case. The use case review gate exists to ensure the spec is good enough before any test is written.

2. **Tests before code.** Every test is written red before the implementation it covers is written.

3. **Preconditions are Given clauses and setup functions.** Never skip them.

4. **Extensions are tests.** Every extension is a test case. An extension without a test is an untested code path waiting to be a production bug.

5. **Postconditions are assertions.** Not documentation. Not comments. Executable assertions.

6. **Domain tests are pure.** No database, no HTTP, no I/O. Fast enough to run on every file save.

7. **Interactor tests use mocks.** The interactor's job is orchestration. Test the call sequence, not the outcomes.

8. **Integration tests own the adapters.** Contract tests verify the adapter implements the port correctly against real infrastructure.

9. **Acceptance tests own the full stack.** They are the outer loop. They define done. They stay red until everything is implemented.

10. **The implementation prompt references the test files.** The AI generates code to make specific, pre-existing tests pass. The tests were derived from the use cases. The use case is therefore enforced at the code level.
