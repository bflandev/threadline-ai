---
name: use-case-to-prompt
description: >
  Use this skill when the user wants to turn a use case into an AI code generation prompt, a test generation prompt, or a data model prompt. Triggers include 'generate code from this use case', 'turn this spec into a prompt', 'create tests from this use case', 'build the implementation prompt', 'what should I send to the AI to build this'. Also use automatically in the agentic pipeline after use-case-reviewer confirms the use case is ready. Do NOT use if the use case has not been reviewed — run use-case-reviewer first.
---

# Use Case to Prompt

Converts a reviewed, ready use case into structured AI generation prompts for implementation, tests, and data models.

## Overview

A use case contains everything an AI needs to generate correct, traceable code — but only if that information is presented in the right structure. This skill formats use case content into prompts that enforce three principles:

1. **Extensions are mandatory.** Most AI generation prompts omit error paths. This skill makes extensions non-negotiable.
2. **Tests are named after steps.** Every generated test references the use case step or extension it covers, creating a permanent traceability link.
3. **Postconditions are assertions.** Postconditions become the test assertions, not supplementary documentation.

## Three prompt types

This skill produces up to three prompts from each use case. Generate all three unless the user specifies otherwise.

---

### Prompt Type 1: Implementation prompt

Instructs the AI to implement the use case as working code.

**Template:**

```
Implement the following use case in [language/framework].

## Use case

Primary actor: [actor]
Goal: [goal]
Preconditions: [preconditions]

Main success scenario:
[numbered steps]

Extensions:
[extension list]

Postconditions:
[postconditions]

## Implementation requirements

1. Implement the main success scenario in full.
2. Implement every extension as an explicit branch or error handler — do not omit any.
3. Each extension must be traceable in the code: add a comment referencing the extension ID (e.g. // UC-02 ext 4a).
4. Preconditions must be enforced at the entry point. If a precondition is not met, handle it explicitly — do not assume it is always true.
5. Postconditions define the expected return value or side effect of a successful run. Implement to these outcomes, not to assumed internal mechanisms.
6. Do not invent behaviour not stated in the use case. If something is ambiguous, output a TODO comment and flag it.

## Framework context

[Insert any relevant: existing models, DB schema, API conventions, auth pattern, relevant existing functions]
```

**Instructions for filling the template:**

- Insert the full use case text verbatim — do not summarise
- Fill the framework context section from whatever codebase context the user provides. If none is provided, omit this section and note that the AI will make assumptions
- If the use case references other use cases (e.g. a precondition is "form has been published" implying UC-01 exists), note those dependencies

---

### Prompt Type 2: Test generation prompt

Instructs the AI to generate a test file where every test maps to a use case step or extension.

**Template:**

```
Generate a test file for the following use case in [language/test framework].

## Use case

[Full use case text]

## Test generation requirements

1. Generate one test per main success scenario step that produces an observable outcome.
2. Generate one test per extension.
3. Name every test using this pattern: test_uc[ID]_step[N]_[description] or test_uc[ID]_ext[Na]_[description].
   Examples:
     test_uc02_step4_required_field_blocks_submission
     test_uc02_ext4a_empty_required_field_highlighted
     test_uc02_ext6a_approval_mode_writes_pending_status
4. Use the postconditions as the assertions for the main success scenario tests.
5. Use the extension outcome as the assertion for each extension test.
6. Generate the test setup (fixtures, mocks, factories) from the preconditions.
7. Do not generate tests for system-internal steps with no observable outcome.

## Test context

[Insert: test framework, mocking library, existing fixture patterns, DB setup approach]
```

**Naming convention details:**

The naming pattern `test_uc[ID]_step[N]_[description]` creates a permanent link between the test and the spec. When a test fails, a developer can look up `UC-02 step 4` in the use case document and understand the expected behaviour immediately, without reading the test implementation.

For extensions: `test_uc02_ext4a_[description]` — use the extension ID exactly as written in the use case.

---

### Prompt Type 3: Data model prompt

Instructs the AI to infer and generate the data models implied by the use case's preconditions and postconditions.

**Template:**

```
Infer the data models required by the following use case and generate them in [language/ORM].

## Use case

[Full use case text]

## Model inference rules

1. Extract entities from: actor names, nouns in preconditions, nouns in postconditions, and nouns in step descriptions.
2. Extract fields from: what is set in steps, what is checked in extensions, what is asserted in postconditions.
3. Extract relationships from: relation fields mentioned in steps, postcondition references to other entities.
4. Extract status fields from: extensions that set a named status (e.g. "record is written with status Pending").
5. Do not infer fields that are not implied by the use case — flag them as potential additions with a comment.

## Output

For each model: name, fields with types, relationships, and any status enums. Then a brief note on what in the use case implied each model.
```

---

## Chaining the prompts

In a full implementation run, the three prompts are used in this order:

1. **Data model prompt first** — establishes the schema that implementation and tests depend on
2. **Implementation prompt second** — uses the schema as context in the framework context section
3. **Test prompt third** — uses both the implementation and schema as context

When chaining, pass the output of each step into the context section of the next prompt. This ensures the tests match the actual implementation, not an assumed one.

## One use case per prompt

Do not combine multiple use cases into a single generation prompt. Each use case should be one generation run. If multiple use cases share models, generate the shared models once (from the use case that defines them) and reference them in subsequent prompts.

## Flag before generating

Before outputting prompts, check:
- Has the use case been reviewed by use-case-reviewer? If not, flag: "This use case has not been reviewed. Unresolved ambiguities may cause the AI to generate incorrect code. Run use-case-reviewer first."
- Are there extensions? If the use case has no extensions, flag: "This use case has no extensions. Either the spec is incomplete, or all failure paths are handled by a guard at the route level. Confirm before generating."
