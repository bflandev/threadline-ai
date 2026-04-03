# From Use Cases to Technical Design
## Clean Architecture and Domain-Driven Design in TypeScript and GoLang

---

## Introduction

This guide explains how to derive a complete Clean Architecture and Domain-Driven Design technical implementation from a set of fully dressed use cases. Every architectural decision is traced back to a specific element of the use case that requires it.

The process is deterministic. Given a well-written use case, the architecture is not a creative exercise — it is a translation. Preconditions become invariants. Extensions become typed errors or domain events. Postconditions become return types and test assertions. Main success scenario steps become interactor method bodies.

The guide uses a FieldBase CMS as the worked example throughout — the same five use cases built in the earlier parts of this series.

Code examples are provided in both TypeScript and GoLang.

---

## Part 1: The Core Insight

Uncle Bob's Clean Architecture defines four concentric layers:

```
┌────────────────────────────────────────────┐
│  Frameworks & Drivers                      │  HTTP, DB, queues, email
│  ┌──────────────────────────────────────┐  │
│  │  Interface Adapters                  │  │  Controllers, presenters, repos
│  │  ┌────────────────────────────────┐  │  │
│  │  │  Use Cases                     │  │  │  Application business rules
│  │  │  ┌──────────────────────────┐  │  │  │
│  │  │  │  Entities                │  │  │  │  Enterprise business rules
│  │  │  └──────────────────────────┘  │  │  │
│  │  └────────────────────────────────┘  │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

**The dependency rule:** source code dependencies point only inward. Nothing in an inner circle knows anything about an outer circle.

Clean Architecture names the layers but leaves the domain layer (Entities) undefined. DDD defines what goes inside it. Together they form a complete design system:

| Layer | Clean Architecture name | DDD concept |
|---|---|---|
| Centre | Entities | Aggregates, Entities, Value Objects, Domain Events |
| Next out | Use Cases | Application Services (Interactors) |
| Next out | Interface Adapters | Controllers, Presenters, Repository implementations |
| Outer | Frameworks & Drivers | HTTP servers, databases, queues, email services |

Your use cases map onto this almost perfectly. The five elements of a fully dressed use case correspond to specific architectural concepts:

| Use case element | Architectural concept |
|---|---|
| Primary actor | Driving port — the primary adapter calls the use case |
| Goal | Public method on the use case interactor |
| Preconditions | Invariant guards inside aggregates |
| Main success scenario steps | The interactor method body |
| Extensions — validation failures | Typed domain errors returned from aggregate methods |
| Extensions — alternate success paths | Domain events raised by aggregates |
| Postconditions | Return type contract and integration test assertions |
| Stakeholders & interests | Non-functional requirements on the adapter layer |

---

## Part 2: Identifying Bounded Contexts

Before writing any code, group your use cases into bounded contexts. A bounded context is a semantic boundary inside which a particular domain model is defined and consistent. The same word can mean different things in different contexts — that is expected, not a problem.

### The five FieldBase use cases

```
UC-01  Create a shared data table     actors: Owner, Collaborator
UC-02  Submit data via a custom form  actors: End User (public)
UC-03  Build a custom page            actors: Editor
UC-04  Manage field-level access      actors: Owner, Collaborator
UC-05  Fork and remix a template      actors: Any authenticated user
```

### Grouping by what changes together

Use cases that share a primary aggregate and speak the same language belong together. Looking at the actors and primary nouns:

- UC-01 and UC-04 both centre on the Table aggregate and involve workspace permissions → **Workspace context**
- UC-02 centres on the Form and Record aggregates, with a public actor → **Submission context**
- UC-03 and UC-05 both centre on Page and Template aggregates, with a publishing concern → **Publishing context**
- All use cases involve authentication and identity → **Identity & Access context** (shared kernel)

### Bounded context map

```
┌─────────────────────────────────────────────────────────────┐
│  Workspace Context                                          │
│  UC-01 (create table), UC-04 (field access)                │
│  Aggregates: Workspace, Table, Field, CollaboratorRole     │
│  Ubiquitous language: schema, field, permission, role      │
└─────────────────────────────────────────────────────────────┘
         │ publishes TableCreated, FieldAccessChanged
         ▼
┌─────────────────────────────────────────────────────────────┐
│  Submission Context                                         │
│  UC-02 (form submission)                                    │
│  Aggregates: Form, Record, SubmissionPolicy                │
│  Ubiquitous language: submission, validation, approval     │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Publishing Context                                         │
│  UC-03 (build page), UC-05 (fork template)                 │
│  Aggregates: Page, Block, Template                         │
│  Ubiquitous language: page, block, slug, publish, fork     │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Identity & Access  — Shared Kernel                        │
│  All use cases                                              │
│  Types: UserId, WorkspaceId, Role, Permission              │
└─────────────────────────────────────────────────────────────┘
```

Bounded contexts communicate through domain events, never by sharing aggregates or importing each other's domain types directly.

---

## Part 3: The Domain Layer

### Value Objects

Value objects are immutable types defined by their value, not their identity. They carry domain meaning and enforce their own validity. Never pass primitives across domain boundaries — wrap them in value objects.

**TypeScript:**
```typescript
// domain/submission/FormId.ts

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
// domain/submission/form_id.go

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

### Aggregates

An aggregate is a consistency boundary. Everything inside it changes together in one transaction. The aggregate root is the only entry point from outside.

#### Deriving aggregates from use case nouns and preconditions

Scan your use cases for:

1. **Nouns in preconditions** — they must exist before the scenario starts, so they are aggregates already in the system
2. **Nouns created in postconditions** — they come into existence as a result, so they are aggregates created by the use case
3. **Nouns that are the subject of extension conditions** — they can be in different states, confirming they are aggregates with lifecycle

For UC-02:
- Precondition: *"A form linked to the target table has been published"* → `Form` aggregate with a published status
- Postcondition: *"A new record exists in the table"* → `Record` aggregate created by submission
- Extension 6a: *"record is written with status Pending"* → `RecordStatus` value object with `Pending` and `Active` states

#### Preconditions become aggregate invariants

The precondition *"A form has been published"* does not become an `if` statement in the HTTP controller or even in the use case interactor. It becomes an invariant guard inside the `Form.submit()` method. This is the critical move: invariant enforcement belongs in the domain, not anywhere else.

**TypeScript:**
```typescript
// domain/submission/Form.ts

export class Form {
  private readonly _events: DomainEvent[] = []

  private constructor(
    public readonly id: FormId,
    public readonly tableId: TableId,
    private readonly fields: FormField[],
    private readonly status: FormStatus,
    private readonly submissionPolicy: SubmissionPolicy,
  ) {}

  static create(
    tableId: TableId,
    fields: FormField[],
    policy: SubmissionPolicy,
  ): Form {
    const form = new Form(
      FormId.generate(),
      tableId,
      fields,
      FormStatus.Draft,
      policy,
    )
    form.raise(new FormCreated(form.id, tableId))
    return form
  }

  static reconstitute(data: FormData): Form {
    return new Form(
      FormId.create(data.id),
      TableId.create(data.tableId),
      data.fields.map(FormField.fromData),
      FormStatus.from(data.status),
      SubmissionPolicy.from(data.policy),
    )
  }

  // UC-02: main success scenario entry point
  // Precondition enforced here — not in the controller or interactor
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

    // UC-02 ext 6a: approval mode determines record status
    const record = Record.create(this.tableId, values, this.submissionPolicy)
    return ok(record)
  }

  private validateFields(values: FieldValues): ValidationResult {
    return this.fields.reduce(
      (result, field) => result.merge(field.validate(values.get(field.id))),
      ValidationResult.empty(),
    )
  }

  domainEvents(): DomainEvent[] { return [...this._events] }
  private raise(event: DomainEvent): void { this._events.push(event) }
}
```

**GoLang:**
```go
// domain/submission/form.go

type Form struct {
    id               FormID
    tableID          TableID
    fields           []FormField
    status           FormStatus
    submissionPolicy SubmissionPolicy
    events           []DomainEvent
}

func NewForm(tableID TableID, fields []FormField, policy SubmissionPolicy) *Form {
    f := &Form{
        id:               GenerateFormID(),
        tableID:          tableID,
        fields:           fields,
        status:           FormStatusDraft,
        submissionPolicy: policy,
    }
    f.raise(&FormCreated{FormID: f.id, TableID: tableID})
    return f
}

func ReconstituteForm(data FormData) *Form {
    return &Form{
        id:               FormID(data.ID),
        tableID:          TableID(data.TableID),
        fields:           reconstitueFields(data.Fields),
        status:           FormStatus(data.Status),
        submissionPolicy: SubmissionPolicy(data.Policy),
    }
}

// UC-02 entry point — precondition enforced here
func (f *Form) Submit(values FieldValues) (*Record, error) {
    // Precondition: form must be published
    if !f.status.IsPublished() {
        return nil, &FormNotPublishedError{FormID: f.id}
    }

    // Ext 4a, 4b: validate all fields
    if errs := f.validateFields(values); len(errs) > 0 {
        return nil, &ValidationError{Errors: errs}
    }

    // Ext 6a: policy determines record status
    return NewRecord(f.tableID, values, f.submissionPolicy), nil
}

func (f *Form) validateFields(values FieldValues) []FieldError {
    var errors []FieldError
    for _, field := range f.fields {
        if err := field.Validate(values.Get(field.ID())); err != nil {
            errors = append(errors, FieldError{FieldID: field.ID(), Err: err})
        }
    }
    return errors
}

func (f *Form) DomainEvents() []DomainEvent { return f.events }
func (f *Form) raise(e DomainEvent)         { f.events = append(f.events, e) }
```

### Extensions become typed errors and domain events

Extensions split into two architectural concepts depending on their nature.

**Error extensions** (validation failures, precondition violations) return typed errors from the aggregate method. Never use generic errors — each extension gets its own type. This allows the use case interactor and the HTTP controller to handle each case explicitly.

**TypeScript — typed error union:**
```typescript
// domain/submission/errors.ts

// One type per extension — never use generic Error in the domain
export type SubmissionError
  = FormNotPublishedError   // precondition violation
  | ValidationError         // UC-02 ext 4a, 4b

export class FormNotPublishedError {
  readonly _tag = 'FormNotPublishedError' as const
  constructor(public readonly formId: FormId) {}
}

export class ValidationError {
  readonly _tag = 'ValidationError' as const
  constructor(public readonly errors: FieldError[]) {}
}

export interface FieldError {
  fieldId: FieldId
  code: 'required' | 'type_mismatch' | 'out_of_range'
  message: string
}
```

**GoLang — sentinel error types:**
```go
// domain/submission/errors.go

type FormNotPublishedError struct {
    FormID FormID
}
func (e *FormNotPublishedError) Error() string {
    return fmt.Sprintf("form %s is not published", e.FormID)
}

type ValidationError struct {
    Errors []FieldError
}
func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed: %d error(s)", len(e.Errors))
}

type FieldError struct {
    FieldID FieldID
    Code    string // "required", "type_mismatch", "out_of_range"
    Message string
}
```

**Alternate success extensions** (UC-02 ext 6a — approval mode routes the record to Pending status and notifies the owner) become domain events raised by the aggregate. The event carries the information needed for the handler to act.

**TypeScript — domain event and handler:**
```typescript
// domain/submission/events.ts

export class RecordCreated implements DomainEvent {
  readonly occurredAt = new Date()
  constructor(
    public readonly recordId: RecordId,
    public readonly tableId: TableId,
    public readonly status: RecordStatus, // 'active' | 'pending'
  ) {}
}

// Inside Record.create() — the event is always raised;
// the status carries whether ext 6a applies
static create(
  tableId: TableId,
  values: FieldValues,
  policy: SubmissionPolicy,
): Record {
  const status = policy.requiresApproval()
    ? RecordStatus.Pending   // UC-02 ext 6a
    : RecordStatus.Active

  const record = new Record(RecordId.generate(), tableId, values, status)
  record.raise(new RecordCreated(record.id, tableId, status))
  return record
}
```

```typescript
// application/submission/NotifyOwnerOnPendingRecord.ts
// UC-02 ext 6a handler — wired at startup, not inside the domain

export class NotifyOwnerOnPendingRecord {
  constructor(private readonly notifier: OwnerNotifier) {}

  async handle(event: RecordCreated): Promise<void> {
    if (!event.status.isPending()) return
    await this.notifier.notifyPendingRecord(event.tableId, event.recordId)
  }
}
```

**GoLang:**
```go
// domain/submission/events.go

type RecordCreated struct {
    RecordID   RecordID
    TableID    TableID
    Status     RecordStatus
    OccurredAt time.Time
}

// application/submission/notify_owner_on_pending.go

type NotifyOwnerOnPendingRecord struct {
    notifier OwnerNotifier // port — defined in application layer
}

func (h *NotifyOwnerOnPendingRecord) Handle(event *RecordCreated) error {
    if event.Status != RecordStatusPending {
        return nil
    }
    return h.notifier.NotifyPendingRecord(event.TableID, event.RecordID)
}
```

---

## Part 4: The Application Layer — Use Case Interactors

The interactor is a direct translation of the main success scenario. Each numbered step in the use case becomes one line or block. Step comments are not optional — they keep the interactor permanently traceable to the spec.

### Rules

- One interactor class or struct per use case
- One public method: `execute(command)` → `Result | error`
- The interactor **orchestrates** — it calls domain methods and repositories. It contains no business logic itself
- No imports from adapters, HTTP packages, or database packages
- Every typed error that surfaces from the domain is propagated to the caller — never swallowed

**TypeScript — UC-02 interactor:**
```typescript
// application/submission/SubmitFormUseCase.ts

export interface SubmitFormCommand {
  formId: string
  values: Record<string, unknown>
}

export interface SubmitFormResult {
  recordId: string
  status: 'active' | 'pending'
}

export class SubmitFormUseCase {
  constructor(
    private readonly forms: FormRepository,
    private readonly records: RecordRepository,
    private readonly events: DomainEventPublisher,
  ) {}

  async execute(
    cmd: SubmitFormCommand,
  ): Promise<Result<SubmitFormResult, SubmissionError>> {

    // Step 1: actor opens form URL — resolved by the adapter; we receive formId

    // Step 2: system renders form fields — handled by a separate query, not this use case

    // Step 3: actor fills in fields and submits — we receive values
    const form = await this.forms.findById(FormId.create(cmd.formId))
    if (!form) {
      return err(new FormNotPublishedError(FormId.create(cmd.formId)))
    }

    // Steps 4–5: validate fields and write record
    // Extensions 4a and 4b surface as typed errors from form.submit()
    const submissionResult = form.submit(FieldValues.from(cmd.values))
    if (submissionResult.isErr()) {
      return submissionResult // propagate ValidationError to adapter
    }

    const record = submissionResult.value
    await this.records.save(record)

    // Step 6: publish events
    // RecordCreated is raised inside Record.create()
    // NotifyOwnerOnPendingRecord fires if status = pending (ext 6a)
    await this.events.publishAll(record.domainEvents())

    // Postcondition: new record exists, submitter has confirmation
    return ok({
      recordId: record.id.value,
      status: record.status.value,
    })
  }
}
```

**GoLang — UC-02 interactor:**
```go
// application/submission/submit_form.go

type SubmitFormCommand struct {
    FormID string
    Values map[string]any
}

type SubmitFormResult struct {
    RecordID string
    Status   string // "active" | "pending"
}

type SubmitFormUseCase struct {
    forms   FormRepository
    records RecordRepository
    events  DomainEventPublisher
}

func (uc *SubmitFormUseCase) Execute(cmd SubmitFormCommand) (*SubmitFormResult, error) {
    // Step 3: load the form
    form, err := uc.forms.FindByID(FormID(cmd.FormID))
    if err != nil {
        return nil, &FormNotPublishedError{FormID: FormID(cmd.FormID)}
    }

    // Steps 4–5: domain validates and creates record
    // Ext 4a, 4b surface as *ValidationError
    record, err := form.Submit(NewFieldValues(cmd.Values))
    if err != nil {
        return nil, err
    }

    if err := uc.records.Save(record); err != nil {
        return nil, err
    }

    // Step 6 + ext 6a: events
    if err := uc.events.PublishAll(record.DomainEvents()); err != nil {
        return nil, err
    }

    return &SubmitFormResult{
        RecordID: string(record.ID()),
        Status:   string(record.Status()),
    }, nil
}
```

Notice what this interactor does not contain: no HTTP, no SQL, no email, no `if status == "pending" { sendEmail() }`. All of that is elsewhere. The interactor is testable with no infrastructure running.

---

## Part 5: Ports — The Application Boundary

Ports are interfaces defined in the application layer and implemented in the adapter layer. They represent the use case's dependencies on the outside world. Every external thing — database, email service, event bus, clock, ID generator — is a port.

This is how the dependency rule is maintained: the application layer defines what it needs; the adapter layer provides it; the application layer never knows how.

**TypeScript — ports for the submission context:**
```typescript
// application/submission/ports.ts

export interface FormRepository {
  findById(id: FormId): Promise<Form | null>
}

export interface RecordRepository {
  save(record: Record): Promise<void>
  findById(id: RecordId): Promise<Record | null>
}

export interface DomainEventPublisher {
  publishAll(events: DomainEvent[]): Promise<void>
}

export interface OwnerNotifier {
  notifyPendingRecord(tableId: TableId, recordId: RecordId): Promise<void>
}
```

**GoLang — ports:**
```go
// application/submission/ports.go

type FormRepository interface {
    FindByID(id FormID) (*Form, error)
}

type RecordRepository interface {
    Save(record *Record) error
    FindByID(id RecordID) (*Record, error)
}

type DomainEventPublisher interface {
    PublishAll(events []DomainEvent) error
}

type OwnerNotifier interface {
    NotifyPendingRecord(tableID TableID, recordID RecordID) error
}
```

Port method signatures use domain types. A repository does not accept a `string` ID — it accepts a `FormId`. This keeps the domain vocabulary consistent across the boundary.

---

## Part 6: Adapters

### Driving adapters — the actor enters here

The HTTP controller is the driving adapter. It translates an HTTP request into a command, calls the use case, and translates the result back into an HTTP response. It is the only place that knows about HTTP status codes, request parsing, or response serialisation.

Each typed domain error maps to a specific HTTP status. The controller is where the extension error taxonomy becomes visible to the outside world.

**TypeScript:**
```typescript
// adapters/http/SubmitFormController.ts

export class SubmitFormController {
  constructor(private readonly useCase: SubmitFormUseCase) {}

  async handle(req: Request, res: Response): Promise<void> {
    const result = await this.useCase.execute({
      formId: req.params.formId,
      values: req.body,
    })

    if (result.isErr()) {
      const error = result.error

      if (error._tag === 'ValidationError') {
        // UC-02 ext 4a, 4b
        res.status(422).json({
          errors: error.errors.map(e => ({
            field: e.fieldId.value,
            code: e.code,
            message: e.message,
          })),
        })
        return
      }

      if (error._tag === 'FormNotPublishedError') {
        res.status(404).json({ message: 'Form not found' })
        return
      }
    }

    const { recordId, status } = result.value

    // UC-02 ext 6a — pending submissions get 202 Accepted
    res.status(status === 'pending' ? 202 : 201).json({ recordId, status })
  }
}
```

**GoLang:**
```go
// adapters/http/submit_form_handler.go

type SubmitFormHandler struct {
    useCase *submission.SubmitFormUseCase
}

func (h *SubmitFormHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    formID := r.PathValue("formId")

    var body map[string]any
    if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
        writeJSON(w, 400, map[string]string{"message": "invalid request body"})
        return
    }

    result, err := h.useCase.Execute(submission.SubmitFormCommand{
        FormID: formID,
        Values: body,
    })

    if err != nil {
        var valErr *submission.ValidationError
        var notFoundErr *submission.FormNotPublishedError

        switch {
        case errors.As(err, &valErr):
            // UC-02 ext 4a, 4b
            writeJSON(w, 422, map[string]any{"errors": valErr.Errors})
        case errors.As(err, &notFoundErr):
            writeJSON(w, 404, map[string]string{"message": "form not found"})
        default:
            writeJSON(w, 500, map[string]string{"message": "internal server error"})
        }
        return
    }

    // UC-02 ext 6a
    statusCode := 201
    if result.Status == "pending" {
        statusCode = 202
    }
    writeJSON(w, statusCode, result)
}

func writeJSON(w http.ResponseWriter, status int, body any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(body)
}
```

### Driven adapters — the use case calls these

The repository adapter implements the repository port using whatever database is chosen. It knows about SQL (or the ORM, or the document store). The domain does not.

**GoLang — Postgres repository:**
```go
// adapters/postgres/form_repository.go

type PostgresFormRepository struct {
    db *sql.DB
}

// Implements submission.FormRepository
func (r *PostgresFormRepository) FindByID(id submission.FormID) (*submission.Form, error) {
    row := r.db.QueryRow(`
        SELECT id, table_id, status, policy, fields_json
        FROM forms
        WHERE id = $1
    `, string(id))

    var rec struct {
        ID         string
        TableID    string
        Status     string
        Policy     string
        FieldsJSON []byte
    }

    if err := row.Scan(&rec.ID, &rec.TableID, &rec.Status, &rec.Policy, &rec.FieldsJSON); err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, nil
        }
        return nil, fmt.Errorf("finding form: %w", err)
    }

    var fields []submission.FormFieldData
    json.Unmarshal(rec.FieldsJSON, &fields)

    return submission.ReconstituteForm(submission.FormData{
        ID:      rec.ID,
        TableID: rec.TableID,
        Status:  rec.Status,
        Policy:  rec.Policy,
        Fields:  fields,
    }), nil
}
```

**TypeScript — email adapter implementing OwnerNotifier port:**
```typescript
// adapters/email/SendGridOwnerNotifier.ts

export class SendGridOwnerNotifier implements OwnerNotifier {
  constructor(
    private readonly client: SendGridClient,
    private readonly tableOwners: TableOwnerLookup,
  ) {}

  async notifyPendingRecord(
    tableId: TableId,
    recordId: RecordId,
  ): Promise<void> {
    const owner = await this.tableOwners.findByTableId(tableId)
    if (!owner) return

    await this.client.send({
      to: owner.email,
      subject: 'New record pending approval',
      text: `Record ${recordId.value} requires your review.`,
    })
  }
}
```

---

## Part 7: The Directory Structure

Derived directly from the bounded contexts and the four architectural layers. Every context gets its own folder at every layer. No cross-context imports in the domain or application layers.

**TypeScript:**
```
src/
├── domain/
│   ├── workspace/
│   │   ├── Table.ts
│   │   ├── Field.ts
│   │   ├── FieldAccessPolicy.ts
│   │   ├── CollaboratorRole.ts
│   │   ├── errors.ts
│   │   └── events/
│   │       ├── TableCreated.ts
│   │       └── FieldAccessChanged.ts
│   ├── submission/
│   │   ├── Form.ts
│   │   ├── Record.ts
│   │   ├── FormField.ts
│   │   ├── FieldValues.ts
│   │   ├── SubmissionPolicy.ts
│   │   ├── RecordStatus.ts
│   │   ├── errors.ts
│   │   └── events/
│   │       └── RecordCreated.ts
│   └── publishing/
│       ├── Page.ts
│       ├── Block.ts
│       ├── Template.ts
│       ├── errors.ts
│       └── events/
│           ├── PagePublished.ts
│           └── TemplateForkd.ts
│
├── application/
│   ├── workspace/
│   │   ├── CreateTableUseCase.ts          ← UC-01
│   │   ├── ManageFieldAccessUseCase.ts    ← UC-04
│   │   └── ports.ts
│   ├── submission/
│   │   ├── SubmitFormUseCase.ts           ← UC-02
│   │   ├── NotifyOwnerOnPendingRecord.ts  ← UC-02 ext 6a handler
│   │   └── ports.ts
│   └── publishing/
│       ├── BuildPageUseCase.ts            ← UC-03
│       ├── ForkTemplateUseCase.ts         ← UC-05
│       └── ports.ts
│
├── adapters/
│   ├── http/
│   │   ├── SubmitFormController.ts
│   │   ├── CreateTableController.ts
│   │   ├── BuildPageController.ts
│   │   └── router.ts
│   ├── postgres/
│   │   ├── FormRepository.ts
│   │   ├── RecordRepository.ts
│   │   ├── TableRepository.ts
│   │   └── PageRepository.ts
│   └── email/
│       └── SendGridOwnerNotifier.ts
│
└── infrastructure/
    ├── DomainEventBus.ts       ← wires events to handlers
    ├── container.ts            ← dependency injection
    ├── database.ts
    └── server.ts
```

**GoLang:**
```
internal/
├── domain/
│   ├── workspace/
│   │   ├── table.go
│   │   ├── field.go
│   │   ├── field_access_policy.go
│   │   ├── errors.go
│   │   └── events.go
│   ├── submission/
│   │   ├── form.go
│   │   ├── record.go
│   │   ├── form_field.go
│   │   ├── field_values.go
│   │   ├── submission_policy.go
│   │   ├── errors.go
│   │   └── events.go
│   └── publishing/
│       ├── page.go
│       ├── block.go
│       ├── template.go
│       ├── errors.go
│       └── events.go
├── application/
│   ├── workspace/
│   │   ├── create_table.go
│   │   ├── manage_field_access.go
│   │   └── ports.go
│   ├── submission/
│   │   ├── submit_form.go
│   │   ├── notify_owner_on_pending.go
│   │   └── ports.go
│   └── publishing/
│       ├── build_page.go
│       ├── fork_template.go
│       └── ports.go
├── adapters/
│   ├── http/
│   │   ├── submit_form_handler.go
│   │   ├── create_table_handler.go
│   │   └── router.go
│   ├── postgres/
│   │   ├── form_repository.go
│   │   ├── record_repository.go
│   │   └── table_repository.go
│   └── email/
│       └── sendgrid_owner_notifier.go
└── infrastructure/
    ├── event_bus.go
    ├── database.go
    └── server.go
```

---

## Part 8: The Dependency Rule

The one rule that makes Clean Architecture work: **source code dependencies point only inward.**

```
infrastructure → adapters → application → domain
                            (ports defined here,
                             implemented in adapters)
```

A violation checklist — if any of these are true, the rule is broken:

- `domain/` imports anything from `application/`, `adapters/`, or `infrastructure/`
- `application/` imports anything from `adapters/` or `infrastructure/`
- `application/` imports a concrete database driver, HTTP package, or email client
- `domain/` imports a concrete database driver, ORM, or any I/O library

The fix is always the same: extract an interface (port) in the inner layer, implement it in the outer layer, and inject the implementation at startup.

### Wiring the dependency graph

The only place that knows about all layers is `infrastructure/`. This is where concrete implementations are instantiated and injected.

**TypeScript:**
```typescript
// infrastructure/container.ts

const db = new PostgresClient(process.env.DATABASE_URL)
const emailClient = new SendGridClient(process.env.SENDGRID_KEY)

// Adapters (implement ports)
const formRepo = new PostgresFormRepository(db)
const recordRepo = new PostgresRecordRepository(db)
const ownerNotifier = new SendGridOwnerNotifier(emailClient, new TableOwnerLookup(db))

// Event bus wires events to handlers
const eventBus = new DomainEventBus()
eventBus.subscribe(RecordCreated, new NotifyOwnerOnPendingRecord(ownerNotifier))

// Use cases (inject ports)
const submitForm = new SubmitFormUseCase(formRepo, recordRepo, eventBus)

// Controllers (inject use cases)
const submitFormController = new SubmitFormController(submitForm)
```

**GoLang:**
```go
// infrastructure/wire.go

func Wire(db *sql.DB, emailClient *sendgrid.Client) *http.ServeMux {
    // Adapters
    formRepo    := postgres.NewFormRepository(db)
    recordRepo  := postgres.NewRecordRepository(db)
    ownerLookup := postgres.NewTableOwnerLookup(db)
    notifier    := email.NewSendGridOwnerNotifier(emailClient, ownerLookup)

    // Event bus
    bus := NewDomainEventBus()
    bus.Subscribe(&submission.RecordCreated{},
        submission.NewNotifyOwnerOnPendingRecord(notifier))

    // Use cases
    submitForm := &submission.SubmitFormUseCase{
        Forms:   formRepo,
        Records: recordRepo,
        Events:  bus,
    }

    // Router
    mux := http.NewServeMux()
    mux.Handle("POST /forms/{formId}/submissions",
        &http_adapters.SubmitFormHandler{UseCase: submitForm})
    return mux
}
```

Nothing in the domain or application layers knows this file exists.

---

## Part 9: Postconditions as Contract Tests

Postconditions define what is observable when a use case ends successfully. They become integration test assertions — not unit tests, because they verify the full chain from use case to persistence.

The pattern for each test:

1. **Arrange** from preconditions — set up the state the use case requires
2. **Act** — call the use case interactor
3. **Assert** from postconditions — verify the observable outcomes

**TypeScript:**
```typescript
// tests/integration/SubmitForm.test.ts

describe('UC-02: Submit data via a custom form', () => {

  it('UC-02 happy path: creates a record and returns confirmation', async () => {
    // Arrange — from precondition: "a form linked to a table has been published"
    const table = await tableRepo.save(Table.create(workspaceId, 'Events'))
    const form = await formRepo.save(
      Form.create(table.id, [nameField, dateField], SubmissionPolicy.noApproval())
    )
    await formRepo.publish(form.id)

    // Act
    const result = await submitFormUseCase.execute({
      formId: form.id.value,
      values: { name: 'React Summit', date: '2025-06-14' },
    })

    // Assert — postcondition 1: "a new record exists in the table"
    expect(result.isOk()).toBe(true)
    const saved = await recordRepo.findById(result.value.recordId)
    expect(saved).not.toBeNull()
    expect(saved!.tableId).toEqual(table.id)

    // Assert — postcondition 2: "submitter receives confirmation"
    expect(result.value.status).toBe('active')
  })

  it('UC-02 ext 4a: required field empty blocks submission', async () => {
    const form = await givenPublishedForm()

    const result = await submitFormUseCase.execute({
      formId: form.id.value,
      values: { date: '2025-06-14' }, // name is missing — required
    })

    expect(result.isErr()).toBe(true)
    expect(result.error._tag).toBe('ValidationError')

    const err = result.error as ValidationError
    expect(err.errors).toContainEqual(
      expect.objectContaining({ code: 'required' })
    )

    // Postcondition: no record was created
    const records = await recordRepo.findByTableId(form.tableId)
    expect(records).toHaveLength(0)
  })

  it('UC-02 ext 6a: approval mode writes pending status and notifies owner', async () => {
    const form = await givenPublishedForm({ requiresApproval: true })

    const result = await submitFormUseCase.execute({
      formId: form.id.value,
      values: { name: 'React Summit', date: '2025-06-14' },
    })

    expect(result.isOk()).toBe(true)

    // Postcondition: record status is pending
    const saved = await recordRepo.findById(result.value.recordId)
    expect(saved!.status.value).toBe('pending')

    // Postcondition: owner was notified
    expect(ownerNotifierSpy.calls).toHaveLength(1)
    expect(ownerNotifierSpy.calls[0].tableId).toEqual(form.tableId)
  })

})
```

**GoLang:**
```go
// tests/integration/submit_form_test.go

func TestSubmitForm_HappyPath(t *testing.T) {
    // Arrange — precondition: published form exists
    db := testDB(t)
    form := givenPublishedForm(t, db, GivenFormOptions{RequiresApproval: false})
    uc := newSubmitFormUseCase(t, db)

    // Act
    result, err := uc.Execute(submission.SubmitFormCommand{
        FormID: string(form.ID()),
        Values: map[string]any{"name": "React Summit", "date": "2025-06-14"},
    })

    // Assert — postcondition 1: record exists
    require.NoError(t, err)
    saved, err := recordRepo(db).FindByID(submission.RecordID(result.RecordID))
    require.NoError(t, err)
    assert.NotNil(t, saved)

    // Assert — postcondition 2: status is active
    assert.Equal(t, "active", result.Status)
}

func TestSubmitForm_Ext4a_RequiredFieldMissing(t *testing.T) {
    db := testDB(t)
    form := givenPublishedForm(t, db, GivenFormOptions{})
    uc := newSubmitFormUseCase(t, db)

    _, err := uc.Execute(submission.SubmitFormCommand{
        FormID: string(form.ID()),
        Values: map[string]any{"date": "2025-06-14"}, // name missing
    })

    var valErr *submission.ValidationError
    require.ErrorAs(t, err, &valErr)

    codes := make([]string, len(valErr.Errors))
    for i, e := range valErr.Errors { codes[i] = e.Code }
    assert.Contains(t, codes, "required")

    // Postcondition: no record created
    records, _ := recordRepo(db).FindByTableID(form.TableID())
    assert.Empty(t, records)
}

func TestSubmitForm_Ext6a_ApprovalMode(t *testing.T) {
    db := testDB(t)
    notifier := &spyOwnerNotifier{}
    form := givenPublishedForm(t, db, GivenFormOptions{RequiresApproval: true})
    uc := newSubmitFormUseCaseWithNotifier(t, db, notifier)

    result, err := uc.Execute(submission.SubmitFormCommand{
        FormID: string(form.ID()),
        Values: map[string]any{"name": "React Summit", "date": "2025-06-14"},
    })

    require.NoError(t, err)
    assert.Equal(t, "pending", result.Status)
    assert.Len(t, notifier.Calls, 1)
}
```

---

## Part 10: Principles in Summary

**Preconditions → aggregate invariants.** Never enforce preconditions in a controller or a use case interactor. They belong in the domain method that requires them.

**Extensions → typed errors or domain events.** Validation failures and precondition violations are typed errors returned from aggregate methods. Alternate success paths and notifications are domain events raised by the aggregate and handled in the application layer.

**Main success scenario → interactor method body.** One step per line or block. Step numbers as comments. The interactor orchestrates — it never contains business logic.

**Ports → interfaces next to the use case.** Every external dependency is an interface defined in the application layer. Adapters implement these interfaces in the outer layer. The inner layers never know about concrete implementations.

**Postconditions → contract test assertions.** Integration tests are arranged from preconditions, acted on by the use case, and asserted from postconditions. Test names reference the use case step or extension ID.

**The dependency rule → always inward.** Domain imports nothing. Application imports domain only. Adapters import application and domain. Infrastructure wires it all together. A violation of this rule is always fixable by extracting a port interface.

---

## Appendix: Skill chain for technical design

This guide corresponds to the `use-case-to-technical-design` skill in the use case skill suite. In the agentic pipeline, it runs after `use-case-reviewer` confirms all use cases are ready, in parallel with the design pipeline (`screen-inventory` → `excalidraw-generator` → `component-inventory` → `figma-spec`).

```
use-case-reviewer (gate)
        │
        ├──────────────────────────────────────────────┐
        ▼                                              ▼
use-case-to-technical-design            screen-inventory
        │                                    │
        ▼                              excalidraw-generator
  bounded-context-map                        │
  aggregate-designs                    component-inventory
  interactor-code                            │
  port-interfaces                       figma-spec
  adapter-sketches                           │
  directory-structure              ──────────┘
  contract-tests                   │
        │                          ▼
        └──────────────► use-case-to-prompt
                                   │
                                   ▼
                           AI generation prompts
                         (implementation + tests)
```

The technical design and the design artefacts (Excalidraw, Figma) are produced from the same use cases. When a use case changes, both pipelines update — that is the point. The use case is the single source of truth for what the product does, how it looks, and how it is built.
